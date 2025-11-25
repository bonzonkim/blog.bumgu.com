---
title: "오픈소스 VictoriaMetrics 에 기여하기"
author: Bumgu
draft: true
date: 2025-11-25T20:42:21+09:00
categories: 
    - DevOps
    - SRE
    - Opensource
tags: 
    - Go
    - HelmChart
    - VictoriaMetrics
slug: "contirubte_victoriaMetrics"
---

# 발단
지난 4월, 회사 로깅 툴을 Loki에서 VictoriaLogs로 교체했다. 꽤 만족스러운 결정이었지만, 팀의 요구사항인 "알람 메시지에 로그 본문 포함"이 안 된다는 것이었다.  
VictoriaLogs는 구조적으로 알람 트리거 시 숫자 값만 반환하기 때문에 로그 텍스트를 알람에 실을 수 없었다. [issue](https://github.com/VictoriaMetrics/VictoriaLogs/issues/81)도 올려보고 방법을 찾아봤지만, 결국 답은 '직접 만들기'뿐이었다.   
아쉬운 대로 간단한 [Webhook Server](https://github.com/bonzonkim/vmalert-webhook)를 구현해 중간에서 로그를 긁어오도록 처리하고 사용했다.  
그런데 나와 같은 불편함을 겪던 몇몇 사람들이 검색을 통해 내 레포지터리를 발견하고 코드를 Fork 해가는 것이었다.  
그 모습을 보고 단순히 땜질 처방으로 남겨두기보다 해결하고 싶어졌다. 그렇게 나는 VictoriaMetrics 공식 레포지터리에(정확히는 그 안의 Vmalert) 해당 기능을 직접 구현해 Contribution하게 되었다.

# 이 기능의 흐름
흐름을 설명하자면,
1. `vmalert`가 알람 조건 평가(rule 작성 시 annotations에 query라는 속성을 추가해야함)  
2. `VictoriaLogs` 에 해당 쿼리(annotations.query)를 가지고 쿼리  
3. 해당하는 리턴(로그본문)을 슬랙 payload로 말아서 slack 에 전송  

그림으로 이미지로 보면 이렇다  
![4](/images/post/29-victorialogs-contribute/4.png)

# 코드 수정
우선 [VictoriaMetrics 공식 레포지터리](https://github.com/VictoriaMetrics/VictoriaMetrics) Fork 해오고, 어느곳에 코드를 넣어야되는지 탐색했다.    


## init.go
`app/vmalert/notifer/init.go`에 새로운 notifier를 추가하기 위해 필요한 flag들을 추가했다.  

```go
vlogsURL    = flag.String("notifier.vlogs.url", "", "URL to VictoriaLogs for querying logs. If set, enables vlogs-webhook integration.")
slackURL    = flag.String("notifier.slack.url", "", "Slack Webhook URL for vlogs-webhook integration.")
vlogIngress = flag.String("notifier.vlogs.ingress", "", "Optional Ingress URL for VictoriaLogs to be used in Slack messages.")
```
위 flag들은 vmalert 실행 시 넘길 수 있는 flag이다. `-notifier.vlogs.url` 식으로!

`vlogsURL`: 검색할 VictoriaLogs URL  
`slackURL`: slack에 보내기 위한 api URL  
`vlogIngress`: 만약 Ingress가 있을 시 알람에 Ingress URL을 포함해서 링크를 클릭하면 해당 쿼리가 검색된 VMUI로 넘어간다.  
이후 원래 Flag로 부터 notifier를 받아오는데 notifiers라는 슬라이스를 생성하고(내가 notifier를 추가했으니까), append하는 식으로 작성했다.


## webhook_vlogs.go
이 파일에서 VictoriaLogs에 쿼리, Slack에 전송을 담당한다.  
먼저 `queryVLogs`함수는 `vmalert`의 `annotations.query`에 담긴 쿼리를 가지고 flag로 넘겨진 `vlogsURL`에 쿼리하고, 로그본문(슬라이스)과 ingressURL을 리턴한다.  
`sendSlack`함수는 넘겨진 로그본문을 가지고 slack payload 형태로 말아서 전송한다.  
`notifier.go`에 이런 interface가 선언되어 있어서 해당 메소드를 가진 구조체를 생성한다.  

```go
// Notifier is a common interface for alert manager provider
type Notifier interface {
	// Send sends the given list of alerts.
	// Returns an error if fails to send the alerts.
	// Must unblock if the given ctx is cancelled.
	Send(ctx context.Context, alerts []Alert, alertLabels [][]prompb.Label, notifierHeaders map[string]string) error
	// Addr returns address where alerts are sent.
	Addr() string
	// LastError returns error, that occured during last attempt to send data
	LastError() string
	// Close is a destructor for the Notifier
	Close()
}
```

```go
// Close is a destructor for the Notifier
func (w *WebhookVLogs) Close() {}

// Addr returns address where alerts are sent
func (w *WebhookVLogs) Addr() string {
	return w.slackURL
}

// LastError returns error, that occured during last attempt to send data
func (w *WebhookVLogs) LastError() string {
	w.mu.Lock()
	defer w.mu.Unlock()
	return w.lastError
}

// Send sends the given list of alerts
func (w *WebhookVLogs) Send(ctx context.Context, alerts []Alert, alertLabels [][]prompb.Label, notifierHeaders map[string]string) error {
	var firstErr error
	w.mu.Lock()
	w.lastError = ""
	w.mu.Unlock()

	for _, alert := range alerts {
		// Check for query annotation
		query, ok := alert.Annotations["query"]
		if !ok || query == "" {
			continue
		}

		logs, logURL, err := w.queryVLogs(ctx, query)
		if err != nil {
			logger.Errorf("failed to query VictoriaLogs for alert %q: %s", alert.Name, err)
			if firstErr == nil {
				firstErr = err
			}
			continue
		}

		if err := w.sendSlack(ctx, alert, logs, logURL); err != nil {
			logger.Errorf("failed to send Slack message for alert %q: %s", alert.Name, err)
			if firstErr == nil {
				firstErr = err
			}
		}
	}

	if firstErr != nil {
		w.mu.Lock()
		w.lastError = firstErr.Error()
		w.mu.Unlock()
	}
	return firstErr
}
```
이렇게 간단하게 해당 interface 를 구현했다.  

코드는 여기서 끝이다. 그만큼 간단한 기능이기 때문에 오랜 시간이 걸리진 않았다.  


# 테스트
이제 테스트를 해야하기 때문에 Slack을 준비하고, container 이미지로 빌드했다.
VictoriaMetrics 레포지터리 루트에서 `make package-vmalert`를 하면 container이미지가 빌드된다.  
테스트를 위한 스크립트, 설정파일 등은 [여기](https://github.com/bonzonkim/victoria-logs-vmalert-feature-integration-test) 에 남겼다.  
`vlog.sh`, `vector.sh`, `vmalert.sh`를 실행하면된다.  
실행하고 브라우저에서 `localhost:9428`로 접속하면 `vector`가 보낸 demo logs들이 나온다.  
![1](/images/post/29-victorialogs-contribute/1.png)
여기서 pretty라는 단어가 많이 나오길래 알람 조건으로 pretty를 걸었다.

```yaml
groups: 
  - name: Test
    type: vlogs
    rules:
      - alert: Test-Alert
        expr: _time:3m * "pretty" | stats count() as err_cnt | filter err_cnt:>0
        for: 0m
        labels:
          severity: critical
          datasource: victoriaLogs
          env: prod
        annotations:
          description: 'Error Log Count: {{ .Value }}'
          query: '_time:5m * "pretty"'
```
위 설정에서 `annotations.query`를 **꼭 적어야한다** 해당 값을 가지고 로그를 검색하기 때문에 꼭 필요한 값이다.  
이후 슬랙을 보면
![2](/images/post/29-victorialogs-contribute/2.png)
"See Logs in VMUI" 라는 링크를 클릭하면 해당 로그가 검색된 VMUI로 이동한다.  

# PR 올리기
의도한대로 작동하는것까지 확인했으니 PR을 올렸다. 영어작문하느라 힘들었다. 이럴때 쓰려고 만들어놓은 문법검사기 [Grammair](https://grammair.co) 를 많이 사용했다.(~~많이써주세요~~)  
PR은 이 링크에서 확인할 수 있다. [링크](https://github.com/VictoriaMetrics/VictoriaMetrics/pull/10070)


![3](/images/post/29-victorialogs-contribute/3.png)
여기서 끝이아니다?!  
이걸 Helm Chart에서도 쉽게 사용할 수 있게 Helm Chart도 PR을 올렸다.

# Helm chart
Helm chart는 읽어만 봤지 사실 한번도 만들어 본 적 없는데 이 기회에 해봤다.  
이 [레포지터리](https://github.com/VictoriaMetrics/helm-charts) 의 `charts/victoria-metrics-alert/templates` 경로에 `_helpers.tpl`과 `values.yaml`만 수정하면 된다.  
내가 nvim에 Go template lsp를 언제 설치해놨는지 모르겠는데 `values.yaml`에 먼저 속성을 정의하면 `_helpers.tpl`에 자동완성이 되길래 `values.yaml`먼저 작성했다.

## values.yaml
`server.notifier` 밑에
```yaml
webhookVlogs:
  enabled: false
  vlogsURL: ""
  slackURL: ""
  ingressURL: ""
```
위 다섯줄만 추가한다. 다른 notifier는 enabled가 없는데 넣을까 말까 하다가 넣었다.

## _helpers.tpl
```templ
{{- define "vmalert.args" -}}
  {{- $Values := (.helm).Values | default .Values -}}
  {{- $app := $Values.server -}}
  {{- $args := default dict -}}
  {{- $_ := set $args "datasource.url" $app.datasource.url -}}
  //# 다른 설정들...
    // notifier.vlogs.url flag 정의
  {{- if .Values.server.notifier.webhookVlogs.enabled -}}
    {{- with .Values.server.notifier.webhookVlogs.vlogsURL -}}
      {{- $_ := set $args "notifier.vlogs.url" . -}}
    {{- end -}}
    // notifier.slack.url flag 정의
    {{- with .Values.server.notifier.webhookVlogs.slackURL -}}
      {{- $_ := set $args "notifier.slack.url" . -}}
    {{- end -}}
    // notifier.vlogs.ingress flag 정의
    {{- with .Values.server.notifier.webhookVlogs.ingressURL -}}
      {{- $_ := set $args "notifier.vlogs.ingress" . -}}
    {{- end -}}
  {{- end -}}
```
이렇게만 하면 Helm chart는 끝이다. 이후 PR을 올렸다.  
해당 PR은 이 링크에서 확인할 수 있다. [링크](https://github.com/VictoriaMetrics/helm-charts/pull/2583)


# 마무리
오랜 버킷리스트였던 오픈소스 기여를 드디어 해봤다.  
이렇게 많이쓰이고, 스타수가 15k가 넘는 대형프로젝트이고, 무엇보다 내가 사용하면서 불편했던 점을 직접 구현하는 경험은 개발자/엔지니어로써 특별한 경험이었다.    
무엇보다 이번 경험은 Go 언어에 대한 나의 노력이 헛되지 않았음을 증명해 준 계기가 되었다.  
비록 아직 PR 검토 단계가 남았지만, 문제를 피하지 않고 코드로 해결책을 제시했다는 점에서 이미 큰 성장을 이뤘다고 생각한다.   
이 코드가 메인 브랜치에 합류하는 순간을 기대하며, 앞으로도 문제를 해결하는 엔지니어가 되어야겠다.  

Code PR: https://github.com/VictoriaMetrics/VictoriaMetrics/pull/10070
helm chart PR: https://github.com/VictoriaMetrics/helm-charts/pull/2583


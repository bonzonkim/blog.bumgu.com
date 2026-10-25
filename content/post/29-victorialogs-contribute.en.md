---
title: "Contributing to the Open-Source VictoriaMetrics"
author: Bumgu
draft: false
date: 2025-11-25T22:50:21+09:00
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

# The origin
Last April, I replaced my company's logging tool, Loki, with VictoriaLogs. It was a decision I was quite happy with, but one of our team's requirements was still unmet: "include the log body in alert messages."  
Because of its design, VictoriaLogs returns only numeric values when an alert is triggered, so it could not include log text in the alert. I opened an [issue](https://github.com/VictoriaMetrics/VictoriaLogs/issues/81) and looked for a way to do it, but in the end the only answer was to "build it myself."  
As a workaround, I implemented a simple [Webhook Server](https://github.com/bonzonkim/vmalert-webhook) to fetch logs in the middle and used that.  
Then I noticed that a few people with the same inconvenience had found my repository through searches and were forking the code.  
Seeing that made me want to solve the problem rather than leave it as a simple patch. So I implemented the feature directly in the official VictoriaMetrics repository (specifically, in Vmalert) and contributed it.

# How the feature works
The flow is as follows:
1. `vmalert` evaluates the alert condition (you need to add a `query` property to the rule's annotations)  
2. Query `VictoriaLogs` using that query (`annotations.query`)  
3. Package the returned result (the log body) as a Slack payload and send it to Slack

Here is the flow shown as a diagram:  
![4](/images/post/29-victorialogs-contribute/4.png)

# Code changes
First, I forked the [official VictoriaMetrics repository](https://github.com/VictoriaMetrics/VictoriaMetrics) and explored where to add the code.    


## init.go
I added the flags needed for the new notifier in `app/vmalert/notifer/init.go`.  

```go
vlogsURL    = flag.String("notifier.vlogs.url", "", "URL to VictoriaLogs for querying logs. If set, enables vlogs-webhook integration.")
slackURL    = flag.String("notifier.slack.url", "", "Slack Webhook URL for vlogs-webhook integration.")
vlogIngress = flag.String("notifier.vlogs.ingress", "", "Optional Ingress URL for VictoriaLogs to be used in Slack messages.")
```
These flags can be passed when running vmalert, for example as `-notifier.vlogs.url`!

`vlogsURL`: VictoriaLogs URL to search  
`slackURL`: API URL for sending to Slack  
`vlogIngress`: If an Ingress is available, include its URL in the alert so clicking the link opens VMUI with the query already run.  
After that, I received the notifier from the existing flags, created a slice called notifiers (since I added a notifier), and appended it.


## webhook_vlogs.go
This file handles querying VictoriaLogs and sending to Slack.  
First, the `queryVLogs` function takes the query in `vmalert`'s `annotations.query`, queries the `vlogsURL` passed as a flag, and returns the log body (slice) and ingressURL.  
The `sendSlack` function takes the returned log body, wraps it in a Slack payload, and sends it.  
An interface like the following is declared in `notifier.go`, so I created a struct with those methods.  

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
I implemented the interface in this straightforward way.  

That's all the code. Since the feature is simple, it didn't take very long.


# Testing
Now I needed to test it, so I prepared Slack and built a container image.
From the VictoriaMetrics repository root, run `make package-vmalert` to build the container image.  
I left the test scripts, configuration files, and so on [here](https://github.com/bonzonkim/victoria-logs-vmalert-feature-integration-test).  
Run `vlog.sh`, `vector.sh`, and `vmalert.sh`.  
After starting them, open `localhost:9428` in a browser to see the demo logs sent by `vector`.  
![1](/images/post/29-victorialogs-contribute/1.png)
I noticed the word pretty appeared often, so I used pretty as the alert condition.

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
You **must include** `annotations.query` in the configuration above. It is required because the logs are searched using that value.  
Then, in Slack:
![2](/images/post/29-victorialogs-contribute/2.png)
Clicking the "See Logs in VMUI" link takes you to VMUI with the relevant logs already searched.

# Submitting the PR
After confirming it worked as intended, I submitted the PR. Writing in English was hard. I made a lot of use of the grammar checker [Grammair](https://grammair.co), which I had made for just this kind of situation. (~~Please use it a lot~~)  
You can view the PR at this link: [link](https://github.com/VictoriaMetrics/VictoriaMetrics/pull/10070)


![3](/images/post/29-victorialogs-contribute/3.png)
But that's not the end?!  
I also submitted a PR for the Helm Chart so this could be used easily with Helm Chart.

# Helm chart
I had only ever read Helm charts and had never actually made one, but I took this opportunity to try it.  
In this [repository](https://github.com/VictoriaMetrics/helm-charts), only `_helpers.tpl` and `values.yaml` under `charts/victoria-metrics-alert/templates` need to be modified.  
I'm not sure when I installed the Go template LSP in nvim, but after defining a property in `values.yaml`, it offered autocomplete in `_helpers.tpl`, so I wrote `values.yaml` first.

## values.yaml
Under `server.notifier`:
```yaml
webhookVlogs:
  enabled: false
  vlogsURL: ""
  slackURL: ""
  ingressURL: ""
```
I added just the lines above. Other notifiers don't have `enabled`, and I debated whether to include it, but decided to do so.

## _helpers.tpl
```templ
{{- define "vmalert.args" -}}
  {{- $Values := (.helm).Values | default .Values -}}
  {{- $app := $Values.server -}}
  {{- $args := default dict -}}
  {{- $_ := set $args "datasource.url" $app.datasource.url -}}
  //# Other settings...
    // Define the notifier.vlogs.url flag
  {{- if .Values.server.notifier.webhookVlogs.enabled -}}
    {{- with .Values.server.notifier.webhookVlogs.vlogsURL -}}
      {{- $_ := set $args "notifier.vlogs.url" . -}}
    {{- end -}}
    // Define the notifier.slack.url flag
    {{- with .Values.server.notifier.webhookVlogs.slackURL -}}
      {{- $_ := set $args "notifier.slack.url" . -}}
    {{- end -}}
    // Define the notifier.vlogs.ingress flag
    {{- with .Values.server.notifier.webhookVlogs.ingressURL -}}
      {{- $_ := set $args "notifier.vlogs.ingress" . -}}
    {{- end -}}
  {{- end -}}
```
That's all there is to the Helm chart. Then I submitted the PR.  
You can view that PR at this link: [link](https://github.com/VictoriaMetrics/helm-charts/pull/2583)


# Conclusion
I finally got to contribute to open source, something that had been on my bucket list for a long time.  
Contributing to a large project that is so widely used and has over 15k stars—and, above all, implementing a solution to a problem I personally encountered while using it—was a special experience as a developer/engineer.  
More than anything, this experience proved that my efforts to learn Go had not been in vain.  
Although the PR still needs to be reviewed, I feel I've already made significant progress by facing the problem and proposing a solution in code rather than avoiding it.  
I look forward to seeing this code merged into the main branch, and I hope to continue becoming an engineer who solves problems.  

Code PR: https://github.com/VictoriaMetrics/VictoriaMetrics/pull/10070
helm chart PR: https://github.com/VictoriaMetrics/helm-charts/pull/2583

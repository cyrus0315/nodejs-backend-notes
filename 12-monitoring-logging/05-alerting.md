# 告警与健康检查

告警和健康检查是保障系统稳定性的最后一道防线。本文深入讲解告警策略和健康检查的设计。

## 目录
- [告警策略](#告警策略)
- [告警渠道](#告警渠道)
- [健康检查](#健康检查)
- [告警规则](#告警规则)
- [On-Call 机制](#on-call-机制)
- [告警降噪](#告警降噪)
- [最佳实践](#最佳实践)
- [面试题](#常见面试题)

---

## 告警策略

### 告警级别

```typescript
enum AlertSeverity {
  CRITICAL = 'critical',  // 严重：立即处理，影响服务
  ERROR = 'error',        // 错误：尽快处理，部分功能受影响
  WARNING = 'warning',    // 警告：需要关注，可能影响未来
  INFO = 'info'          // 信息：仅通知
}

interface Alert {
  severity: AlertSeverity;
  title: string;
  message: string;
  tags: Record<string, string>;
  timestamp: Date;
  source: string;
}
```

### 告警条件

```typescript
// 阈值告警
interface ThresholdAlert {
  metric: string;
  operator: '>' | '<' | '>=' | '<=' | '==' | '!=';
  threshold: number;
  duration: number; // 持续时间（秒）
}

// 变化率告警
interface RateAlert {
  metric: string;
  change: number; // 变化率（%）
  period: number; // 观察周期（秒）
}

// 异常检测告警
interface AnomalyAlert {
  metric: string;
  algorithm: 'zscore' | 'iqr' | 'prophet';
  sensitivity: number; // 敏感度
}
```

---

## 告警渠道

### Slack

```typescript
import fetch from 'node-fetch';

interface SlackMessage {
  text: string;
  attachments?: any[];
  blocks?: any[];
}

class SlackNotifier {
  constructor(private webhookUrl: string) {}

  async send(alert: Alert): Promise<void> {
    const color = this.getColor(alert.severity);
    
    const message: SlackMessage = {
      text: `🚨 ${alert.title}`,
      attachments: [{
        color,
        title: alert.title,
        text: alert.message,
        fields: Object.entries(alert.tags).map(([key, value]) => ({
          title: key,
          value,
          short: true
        })),
        footer: alert.source,
        ts: Math.floor(alert.timestamp.getTime() / 1000)
      }]
    };

    await fetch(this.webhookUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(message)
    });
  }

  private getColor(severity: AlertSeverity): string {
    switch (severity) {
      case AlertSeverity.CRITICAL:
        return 'danger';
      case AlertSeverity.ERROR:
        return 'warning';
      case AlertSeverity.WARNING:
        return '#ffcc00';
      default:
        return 'good';
    }
  }
}

// 使用
const slack = new SlackNotifier(process.env.SLACK_WEBHOOK_URL!);

await slack.send({
  severity: AlertSeverity.CRITICAL,
  title: 'High Error Rate',
  message: 'Error rate exceeded 5% in the last 5 minutes',
  tags: {
    environment: 'production',
    service: 'api',
    error_rate: '7.2%'
  },
  timestamp: new Date(),
  source: 'Prometheus'
});
```

### Email

```typescript
import nodemailer from 'nodemailer';

class EmailNotifier {
  private transporter: nodemailer.Transporter;

  constructor() {
    this.transporter = nodemailer.createTransporter({
      host: process.env.SMTP_HOST,
      port: parseInt(process.env.SMTP_PORT || '587'),
      secure: false,
      auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASS
      }
    });
  }

  async send(alert: Alert, recipients: string[]): Promise<void> {
    const html = this.renderHtml(alert);

    await this.transporter.sendMail({
      from: process.env.ALERT_FROM_EMAIL,
      to: recipients.join(','),
      subject: `[${alert.severity.toUpperCase()}] ${alert.title}`,
      html
    });
  }

  private renderHtml(alert: Alert): string {
    const color = alert.severity === AlertSeverity.CRITICAL ? '#ff0000' : '#ff9900';

    return `
      <div style="font-family: Arial, sans-serif;">
        <div style="background-color: ${color}; color: white; padding: 20px;">
          <h2>${alert.title}</h2>
        </div>
        <div style="padding: 20px;">
          <p>${alert.message}</p>
          <table>
            ${Object.entries(alert.tags).map(([key, value]) => `
              <tr>
                <td style="font-weight: bold; padding: 5px;">${key}:</td>
                <td style="padding: 5px;">${value}</td>
              </tr>
            `).join('')}
          </table>
          <p style="color: #666; margin-top: 20px;">
            ${alert.source} | ${alert.timestamp.toISOString()}
          </p>
        </div>
      </div>
    `;
  }
}
```

### SMS（Twilio）

```typescript
import twilio from 'twilio';

class SMSNotifier {
  private client: twilio.Twilio;

  constructor() {
    this.client = twilio(
      process.env.TWILIO_ACCOUNT_SID,
      process.env.TWILIO_AUTH_TOKEN
    );
  }

  async send(alert: Alert, phoneNumbers: string[]): Promise<void> {
    const message = `[${alert.severity.toUpperCase()}] ${alert.title}\n${alert.message}`;

    for (const phone of phoneNumbers) {
      await this.client.messages.create({
        body: message,
        from: process.env.TWILIO_PHONE_NUMBER,
        to: phone
      });
    }
  }
}
```

### PagerDuty

```typescript
class PagerDutyNotifier {
  constructor(private routingKey: string) {}

  async send(alert: Alert): Promise<void> {
    await fetch('https://events.pagerduty.com/v2/enqueue', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        routing_key: this.routingKey,
        event_action: 'trigger',
        dedup_key: this.generateDedupKey(alert),
        payload: {
          summary: alert.title,
          severity: this.mapSeverity(alert.severity),
          source: alert.source,
          timestamp: alert.timestamp.toISOString(),
          custom_details: {
            message: alert.message,
            tags: alert.tags
          }
        }
      })
    });
  }

  private generateDedupKey(alert: Alert): string {
    // 用于去重相同的告警
    return `${alert.source}-${alert.title}-${JSON.stringify(alert.tags)}`;
  }

  private mapSeverity(severity: AlertSeverity): string {
    const mapping = {
      [AlertSeverity.CRITICAL]: 'critical',
      [AlertSeverity.ERROR]: 'error',
      [AlertSeverity.WARNING]: 'warning',
      [AlertSeverity.INFO]: 'info'
    };
    return mapping[severity];
  }

  async resolve(dedupKey: string): Promise<void> {
    await fetch('https://events.pagerduty.com/v2/enqueue', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        routing_key: this.routingKey,
        event_action: 'resolve',
        dedup_key: dedupKey
      })
    });
  }
}
```

### 多渠道告警

```typescript
class AlertManager {
  private notifiers: Map<string, Notifier> = new Map();

  registerNotifier(name: string, notifier: Notifier): void {
    this.notifiers.set(name, notifier);
  }

  async sendAlert(alert: Alert, channels: string[]): Promise<void> {
    const promises = channels.map(async (channel) => {
      const notifier = this.notifiers.get(channel);
      if (!notifier) {
        console.warn(`Notifier ${channel} not found`);
        return;
      }

      try {
        await notifier.send(alert);
      } catch (error) {
        console.error(`Failed to send alert via ${channel}:`, error);
      }
    });

    await Promise.allSettled(promises);
  }

  // 根据严重程度选择渠道
  async sendAlertBySeverity(alert: Alert): Promise<void> {
    let channels: string[];

    switch (alert.severity) {
      case AlertSeverity.CRITICAL:
        channels = ['pagerduty', 'slack', 'sms'];
        break;
      case AlertSeverity.ERROR:
        channels = ['slack', 'email'];
        break;
      case AlertSeverity.WARNING:
        channels = ['slack'];
        break;
      default:
        channels = [];
    }

    await this.sendAlert(alert, channels);
  }
}

// 使用
const alertManager = new AlertManager();

alertManager.registerNotifier('slack', new SlackNotifier(process.env.SLACK_WEBHOOK_URL!));
alertManager.registerNotifier('email', new EmailNotifier());
alertManager.registerNotifier('sms', new SMSNotifier());
alertManager.registerNotifier('pagerduty', new PagerDutyNotifier(process.env.PAGERDUTY_ROUTING_KEY!));

// 发送告警
await alertManager.sendAlertBySeverity({
  severity: AlertSeverity.CRITICAL,
  title: 'Service Down',
  message: 'API service is not responding',
  tags: {
    environment: 'production',
    service: 'api'
  },
  timestamp: new Date(),
  source: 'Kubernetes'
});
```

---

## 健康检查

### 基础健康检查

```typescript
import express from 'express';

const app = express();

// Liveness Probe（存活探针）
app.get('/health/live', (req, res) => {
  // 简单返回 200，表示进程还活着
  res.status(200).json({ status: 'ok' });
});

// Readiness Probe（就绪探针）
app.get('/health/ready', async (req, res) => {
  try {
    // 检查依赖服务
    await Promise.all([
      checkDatabase(),
      checkRedis(),
      checkExternalAPI()
    ]);

    res.status(200).json({
      status: 'ready',
      timestamp: new Date().toISOString()
    });
  } catch (error) {
    res.status(503).json({
      status: 'not_ready',
      error: error.message,
      timestamp: new Date().toISOString()
    });
  }
});

async function checkDatabase(): Promise<void> {
  await prisma.$queryRaw`SELECT 1`;
}

async function checkRedis(): Promise<void> {
  await redis.ping();
}

async function checkExternalAPI(): Promise<void> {
  const response = await fetch('https://api.example.com/health', {
    timeout: 3000
  });
  
  if (!response.ok) {
    throw new Error('External API is down');
  }
}
```

### 详细健康检查

```typescript
interface HealthCheckResult {
  status: 'healthy' | 'degraded' | 'unhealthy';
  timestamp: string;
  version: string;
  uptime: number;
  checks: Record<string, CheckResult>;
}

interface CheckResult {
  status: 'pass' | 'fail' | 'warn';
  message?: string;
  duration?: number;
  metadata?: Record<string, any>;
}

class HealthChecker {
  private checks: Map<string, () => Promise<CheckResult>> = new Map();

  register(name: string, check: () => Promise<CheckResult>): void {
    this.checks.set(name, check);
  }

  async check(): Promise<HealthCheckResult> {
    const results: Record<string, CheckResult> = {};
    const promises: Promise<void>[] = [];

    for (const [name, check] of this.checks.entries()) {
      promises.push(
        (async () => {
          const start = Date.now();
          try {
            results[name] = await Promise.race([
              check(),
              this.timeout(5000) // 5 秒超时
            ]);
            results[name].duration = Date.now() - start;
          } catch (error) {
            results[name] = {
              status: 'fail',
              message: error.message,
              duration: Date.now() - start
            };
          }
        })()
      );
    }

    await Promise.all(promises);

    const status = this.determineOverallStatus(results);

    return {
      status,
      timestamp: new Date().toISOString(),
      version: process.env.APP_VERSION || 'unknown',
      uptime: process.uptime(),
      checks: results
    };
  }

  private determineOverallStatus(
    checks: Record<string, CheckResult>
  ): 'healthy' | 'degraded' | 'unhealthy' {
    const statuses = Object.values(checks).map(c => c.status);

    if (statuses.every(s => s === 'pass')) {
      return 'healthy';
    }

    if (statuses.some(s => s === 'fail')) {
      return 'unhealthy';
    }

    return 'degraded';
  }

  private async timeout(ms: number): Promise<CheckResult> {
    await new Promise((resolve) => setTimeout(resolve, ms));
    return {
      status: 'fail',
      message: 'Health check timeout'
    };
  }
}

// 使用
const healthChecker = new HealthChecker();

// 数据库检查
healthChecker.register('database', async () => {
  try {
    const start = Date.now();
    await prisma.$queryRaw`SELECT 1`;
    const duration = Date.now() - start;

    return {
      status: duration < 100 ? 'pass' : 'warn',
      message: duration < 100 ? 'Database is healthy' : 'Database is slow',
      metadata: { latency_ms: duration }
    };
  } catch (error) {
    return {
      status: 'fail',
      message: error.message
    };
  }
});

// Redis 检查
healthChecker.register('redis', async () => {
  try {
    await redis.ping();
    return {
      status: 'pass',
      message: 'Redis is healthy'
    };
  } catch (error) {
    return {
      status: 'fail',
      message: error.message
    };
  }
});

// 磁盘空间检查
healthChecker.register('disk', async () => {
  const diskUsage = await checkDiskUsage();
  
  if (diskUsage < 80) {
    return {
      status: 'pass',
      message: 'Disk space is sufficient',
      metadata: { usage_percent: diskUsage }
    };
  } else if (diskUsage < 90) {
    return {
      status: 'warn',
      message: 'Disk space is running low',
      metadata: { usage_percent: diskUsage }
    };
  } else {
    return {
      status: 'fail',
      message: 'Disk space is critically low',
      metadata: { usage_percent: diskUsage }
    };
  }
});

// 内存检查
healthChecker.register('memory', async () => {
  const used = process.memoryUsage();
  const heapUsedPercent = (used.heapUsed / used.heapTotal) * 100;

  if (heapUsedPercent < 80) {
    return {
      status: 'pass',
      message: 'Memory usage is normal',
      metadata: {
        heap_used_mb: Math.round(used.heapUsed / 1024 / 1024),
        heap_total_mb: Math.round(used.heapTotal / 1024 / 1024),
        usage_percent: Math.round(heapUsedPercent)
      }
    };
  } else {
    return {
      status: 'warn',
      message: 'Memory usage is high',
      metadata: {
        heap_used_mb: Math.round(used.heapUsed / 1024 / 1024),
        heap_total_mb: Math.round(used.heapTotal / 1024 / 1024),
        usage_percent: Math.round(heapUsedPercent)
      }
    };
  }
});

// 健康检查端点
app.get('/health', async (req, res) => {
  const result = await healthChecker.check();
  
  const statusCode = result.status === 'healthy' ? 200 :
                     result.status === 'degraded' ? 200 : 503;
  
  res.status(statusCode).json(result);
});
```

### Kubernetes 健康检查

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: app
          image: my-app:1.0.0
          ports:
            - containerPort: 3000
          
          # 存活探针
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          
          # 就绪探针
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          
          # 启动探针
          startupProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 30
```

---

## 告警规则

### Prometheus 告警

```yaml
# alerts.yml
groups:
  - name: api_alerts
    interval: 30s
    rules:
      # 服务不可用
      - alert: ServiceDown
        expr: up{job="api"} == 0
        for: 1m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Service {{ $labels.instance }} is down"
          description: "{{ $labels.instance }} has been down for more than 1 minute"
          runbook_url: "https://wiki.example.com/runbooks/service-down"

      # 高错误率
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[5m]))
            / 
            sum(rate(http_requests_total[5m]))
          ) > 0.05
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"
          dashboard_url: "https://grafana.example.com/d/errors"

      # 高延迟
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route)
          ) > 1
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "High latency on {{ $labels.route }}"
          description: "P95 latency is {{ $value }}s (threshold: 1s)"

      # CPU 使用率高
      - alert: HighCPU
        expr: |
          100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 10m
        labels:
          severity: warning
          team: infra
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | humanize }}% (threshold: 80%)"

      # 内存使用率高
      - alert: HighMemory
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels:
          severity: critical
          team: infra
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | humanize }}% (threshold: 90%)"

      # 磁盘空间不足
      - alert: LowDiskSpace
        expr: |
          (node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs"} / node_filesystem_size_bytes) * 100 < 10
        for: 5m
        labels:
          severity: warning
          team: infra
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Only {{ $value | humanize }}% disk space left (threshold: 10%)"

      # 数据库连接池耗尽
      - alert: DatabasePoolExhausted
        expr: db_pool_idle_connections == 0
        for: 2m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Database connection pool exhausted"
          description: "No idle database connections available"

      # 队列积压
      - alert: QueueBacklog
        expr: queue_size > 1000
        for: 10m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Queue backlog detected"
          description: "Queue {{ $labels.queue_name }} has {{ $value }} pending items"
```

### Alertmanager 配置

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'

# 路由规则
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  
  routes:
    # 严重告警 -> PagerDuty + Slack
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true
    
    - match:
        severity: critical
      receiver: 'slack-critical'
    
    # 警告 -> Slack
    - match:
        severity: warning
      receiver: 'slack-warnings'
    
    # 基础设施团队
    - match:
        team: infra
      receiver: 'slack-infra'

# 接收器
receivers:
  - name: 'default'
    slack_configs:
      - channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
        description: '{{ .GroupLabels.alertname }}'
        details:
          firing: '{{ .Alerts.Firing | len }}'
          resolved: '{{ .Alerts.Resolved | len }}'

  - name: 'slack-critical'
    slack_configs:
      - channel: '#critical-alerts'
        color: 'danger'
        title: '🚨 {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#warnings'
        color: 'warning'
        title: '⚠️ {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'slack-infra'
    slack_configs:
      - channel: '#infra-alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

# 抑制规则
inhibit_rules:
  # 如果服务宕机，抑制其他告警
  - source_match:
      alertname: 'ServiceDown'
    target_match_re:
      alertname: 'High.*'
    equal: ['instance']
```

---

## On-Call 机制

### 轮值表

```typescript
interface OnCallSchedule {
  id: string;
  team: string;
  rotations: Rotation[];
}

interface Rotation {
  id: string;
  start: Date;
  end: Date;
  primary: User;
  secondary?: User;
}

interface User {
  id: string;
  name: string;
  email: string;
  phone: string;
  timezone: string;
}

class OnCallManager {
  async getCurrentOnCall(team: string): Promise<Rotation | null> {
    const schedule = await this.getSchedule(team);
    const now = new Date();

    return schedule.rotations.find(
      rotation => now >= rotation.start && now <= rotation.end
    ) || null;
  }

  async escalate(alert: Alert, team: string): Promise<void> {
    const rotation = await this.getCurrentOnCall(team);

    if (!rotation) {
      throw new Error('No on-call person found');
    }

    // 1. 通知主要值班人员
    await this.notifyUser(rotation.primary, alert, 'primary');

    // 2. 等待 5 分钟
    await this.sleep(5 * 60 * 1000);

    // 3. 如果未确认，升级到备用人员
    if (!await this.isAcknowledged(alert)) {
      if (rotation.secondary) {
        await this.notifyUser(rotation.secondary, alert, 'escalated');
      } else {
        // 升级到团队负责人
        await this.notifyTeamLead(team, alert);
      }
    }
  }

  private async notifyUser(user: User, alert: Alert, level: string): Promise<void> {
    // 发送多渠道通知
    await Promise.all([
      this.sendSMS(user.phone, alert),
      this.sendEmail(user.email, alert),
      this.callPhone(user.phone, alert) // 严重告警时打电话
    ]);
  }
}
```

---

## 告警降噪

### 告警聚合

```typescript
class AlertAggregator {
  private buffer: Map<string, Alert[]> = new Map();
  private flushInterval = 60000; // 1 分钟

  constructor() {
    setInterval(() => this.flush(), this.flushInterval);
  }

  add(alert: Alert): void {
    const key = this.getGroupKey(alert);
    
    if (!this.buffer.has(key)) {
      this.buffer.set(key, []);
    }
    
    this.buffer.get(key)!.push(alert);
  }

  private getGroupKey(alert: Alert): string {
    // 按服务和告警类型分组
    return `${alert.tags.service}-${alert.title}`;
  }

  private async flush(): Promise<void> {
    for (const [key, alerts] of this.buffer.entries()) {
      if (alerts.length === 0) continue;

      if (alerts.length === 1) {
        // 单个告警，正常发送
        await this.sendAlert(alerts[0]);
      } else {
        // 多个告警，聚合发送
        await this.sendAggregatedAlert(key, alerts);
      }
    }

    this.buffer.clear();
  }

  private async sendAggregatedAlert(key: string, alerts: Alert[]): Promise<void> {
    const aggregated: Alert = {
      severity: this.getHighestSeverity(alerts),
      title: `${alerts.length} alerts: ${key}`,
      message: this.aggregateMessages(alerts),
      tags: alerts[0].tags,
      timestamp: new Date(),
      source: alerts[0].source
    };

    await alertManager.sendAlert(aggregated, ['slack']);
  }

  private getHighestSeverity(alerts: Alert[]): AlertSeverity {
    const severityOrder = {
      [AlertSeverity.CRITICAL]: 4,
      [AlertSeverity.ERROR]: 3,
      [AlertSeverity.WARNING]: 2,
      [AlertSeverity.INFO]: 1
    };

    return alerts.reduce((highest, alert) => {
      return severityOrder[alert.severity] > severityOrder[highest]
        ? alert.severity
        : highest;
    }, AlertSeverity.INFO);
  }

  private aggregateMessages(alerts: Alert[]): string {
    const counts = new Map<string, number>();
    
    for (const alert of alerts) {
      counts.set(alert.message, (counts.get(alert.message) || 0) + 1);
    }

    return Array.from(counts.entries())
      .map(([message, count]) => `${message} (${count}x)`)
      .join('\n');
  }
}
```

### 告警静默

```typescript
interface SilenceRule {
  id: string;
  matchers: Record<string, string>;
  start: Date;
  end: Date;
  creator: string;
  comment: string;
}

class AlertSilencer {
  private silences: SilenceRule[] = [];

  addSilence(rule: SilenceRule): void {
    this.silences.push(rule);
  }

  removeSilence(id: string): void {
    this.silences = this.silences.filter(s => s.id !== id);
  }

  shouldSilence(alert: Alert): boolean {
    const now = new Date();

    return this.silences.some(silence => {
      // 检查时间范围
      if (now < silence.start || now > silence.end) {
        return false;
      }

      // 检查匹配条件
      return Object.entries(silence.matchers).every(([key, value]) => {
        return alert.tags[key] === value;
      });
    });
  }
}

// 使用
const silencer = new AlertSilencer();

// 维护期间静默告警
silencer.addSilence({
  id: 'maintenance-1',
  matchers: {
    service: 'api',
    environment: 'production'
  },
  start: new Date('2024-01-01T02:00:00Z'),
  end: new Date('2024-01-01T04:00:00Z'),
  creator: 'admin',
  comment: 'Database maintenance'
});

// 发送告警前检查
async function sendAlert(alert: Alert): Promise<void> {
  if (silencer.shouldSilence(alert)) {
    console.log('Alert silenced:', alert.title);
    return;
  }

  await alertManager.sendAlert(alert, ['slack']);
}
```

---

## 最佳实践

### 1. 告警原则

```typescript
// ✅ 好的告警
// - 可操作：收到告警后知道该做什么
// - 有意义：告警真的表示有问题
// - 及时：问题发生时立即告警
// - 准确：低误报率

// ❌ 不好的告警
// - 太多：告警疲劳
// - 太少：问题未被发现
// - 模糊：不知道问题在哪
// - 延迟：问题已经过去才告警
```

### 2. Runbook

```typescript
// 每个告警都应该有对应的 Runbook
interface Runbook {
  alert: string;
  severity: AlertSeverity;
  description: string;
  impact: string;
  diagnosis: string[];
  resolution: string[];
  escalation: string;
}

const runbooks: Map<string, Runbook> = new Map([
  ['HighErrorRate', {
    alert: 'HighErrorRate',
    severity: AlertSeverity.CRITICAL,
    description: 'API 错误率超过 5%',
    impact: '影响用户体验，部分请求失败',
    diagnosis: [
      '1. 检查 Grafana Dashboard 查看错误分布',
      '2. 查看 Sentry 最新错误',
      '3. 检查最近的部署',
      '4. 查看数据库性能'
    ],
    resolution: [
      '1. 如果是新部署导致，回滚到上一个版本',
      '2. 如果是数据库问题，检查慢查询',
      '3. 如果是外部服务问题，启用降级方案',
      '4. 修复代码后重新部署'
    ],
    escalation: '如果 30 分钟内无法解决，升级到 Team Lead'
  }]
]);
```

### 3. 告警测试

```typescript
// 定期测试告警系统
async function testAlertSystem(): Promise<void> {
  const testAlert: Alert = {
    severity: AlertSeverity.WARNING,
    title: 'Alert System Test',
    message: 'This is a test alert',
    tags: {
      test: 'true',
      environment: 'test'
    },
    timestamp: new Date(),
    source: 'test'
  };

  await alertManager.sendAlert(testAlert, ['slack']);
  
  console.log('Test alert sent successfully');
}

// 每周测试一次
setInterval(testAlertSystem, 7 * 24 * 60 * 60 * 1000);
```

---

## 常见面试题

### 1. 如何设计告警系统？

**要点**：

1. **告警级别**：Critical、Error、Warning、Info
2. **告警渠道**：Slack、Email、SMS、PagerDuty
3. **告警路由**：根据级别和团队路由
4. **告警聚合**：避免告警风暴
5. **告警静默**：维护期间静默
6. **告警升级**：未确认时升级
7. **Runbook**：每个告警都有处理文档

### 2. 如何避免告警疲劳？

**方法**：

1. **减少噪音**：
   - 提高告警阈值
   - 增加持续时间
   - 过滤已知问题

2. **告警聚合**：
   - 相同类型告警合并
   - 批量通知

3. **告警优先级**：
   - 只有重要告警才通知
   - 不重要的记录日志

4. **自动化**：
   - 自动修复常见问题
   - 自动静默重复告警

5. **定期审查**：
   - 检查告警有效性
   - 删除无用告警

### 3. Liveness vs Readiness？

| 探针 | 作用 | 失败后果 |
|------|------|---------|
| **Liveness** | 检查进程是否存活 | 重启容器 |
| **Readiness** | 检查是否可以接收流量 | 从负载均衡移除 |
| **Startup** | 检查是否启动完成 | 等待或重启 |

**示例**：
- Liveness：检查进程是否死锁
- Readiness：检查数据库连接是否正常
- Startup：检查初始化是否完成

### 4. 如何处理告警风暴？

**步骤**：

1. **识别根因**：
   - 查看告警时间线
   - 识别最早的告警

2. **临时静默**：
   - 静默衍生告警
   - 保留根因告警

3. **快速修复**：
   - 优先修复根因
   - 或启用降级方案

4. **事后分析**：
   - 为什么会产生告警风暴
   - 如何改进告警规则

### 5. On-Call 最佳实践？

**实践**：

1. **明确轮值**：
   - 清晰的轮值表
   - 自动化轮值提醒

2. **合理负载**：
   - 不要太频繁
   - 合理的休息时间

3. **工具支持**：
   - PagerDuty 等 On-Call 工具
   - 移动应用

4. **Runbook**：
   - 每个告警都有处理文档
   - 定期更新

5. **事后总结**：
   - 记录处理过程
   - 改进告警和流程

---

## 总结

### 告警系统架构

```
告警源（Prometheus、Sentry、自定义）
    ↓
告警聚合与去重
    ↓
告警路由（按级别、团队）
    ↓
告警渠道（Slack、Email、SMS、PagerDuty）
    ↓
On-Call 升级
```

### 实践检查清单

- [ ] 设置健康检查端点
- [ ] 配置告警规则
- [ ] 集成告警渠道
- [ ] 编写 Runbook
- [ ] 设置 On-Call 轮值
- [ ] 实施告警聚合
- [ ] 配置告警静默
- [ ] 定期测试告警系统
- [ ] 审查告警有效性
- [ ] 优化告警阈值

---

**上一篇**：[错误追踪](./04-error-tracking.md)  
**返回目录**：[监控与日志](./README.md)


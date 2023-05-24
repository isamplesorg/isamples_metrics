# metrics
Docker config for iSamples Metrics packages deployed in the cloud

## plausible.io

# iSamples analytics setup using plausible.io

## Setup
The general instructions were to follow the self-hosting guide on plausible's site, as described here: https://plausible.io/docs/self-hosting

1. git clone https://github.com/plausible/hosting
2. cd to that directory and edit `plausible-conf.env`
```
ADMIN_USER_EMAIL=danny.mandel@gmail.com
ADMIN_USER_NAME=isamples
ADMIN_USER_PWD=<password>
BASE_URL=https://mars.cyverse.org/metrics/
SECRET_KEY_BASE=eCp4Vj5TZTRFlodkctIDpx+Oymzib3uxaY6glSqGS1RDlcxoJc7rof5l2M5zxqPPJRvsLx9efjt9f4ZxDYAoTQ==
PORT=8788
```
3. Make sure to edit the port in `docker-compose.yml` to match the port in the config file:
```
mandeld@SBS-7448 plausible-hosting % git diff
diff --git a/docker-compose.yml b/docker-compose.yml
index a4f9f2d..caf4141 100644
--- a/docker-compose.yml
+++ b/docker-compose.yml
@@ -33,7 +33,7 @@ services:
       - plausible_events_db
       - mail
     ports:
-      - 8000:8000
+      - 8788:8788
```
4. Next, set up a linux service to start/stop the plausible instance on the host machine:
```
dannymandel@mars:~$ cat /etc/systemd/system/plausible-io.service 
    [Unit]
    Description=Docker Compose plausible.io Application Service
    Requires=docker.service
    After=docker.service

    [Service]
    Type=oneshot
    RemainAfterExit=yes
    WorkingDirectory=/home/isamples/plausible_hosting
    ExecStart=/usr/bin/docker-compose up -d
    ExecStop=/usr/bin/docker-compose down 
    TimeoutStartSec=0

    [Install]
    WantedBy=multi-user.target  
```
5. Reload systemctl daemon, enable the plausible service, and start it:
```
sudo systemctl daemon-reload
sudo systemctl enable plausible-io
sudo systemctl start plausible-io
```
6. Configure nginx to redirect https://metrics.isample.xyz/ traffic to plausible -- this is in `/etc/nginx/sites-enabled/default`:
```
server {
	root /var/www/html;

	index index.html index.htm index.nginx-debian.html;

	server_name metrics.isample.xyz;	

    location / {
      proxy_set_header Host $http_host;
      #proxy_set_header Host $host;
      #proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_set_header X-Forwarded-Scheme $scheme;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "Upgrade";
      # proxy_set_header Connection $connection_upgrade;
      proxy_set_header   X-Real-IP $remote_addr;
      proxy_set_header   X-Forwarded-Host $server_name;
      proxy_redirect off;
      proxy_buffering off;
      proxy_http_version 1.1;
      proxy_pass http://localhost:8788;
      }

	listen 443;

    ssl_certificate /etc/letsencrypt/live/metrics.isample.xyz/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/metrics.isample.xyz/privkey.pem; # managed by Certbot
}
```
7. Configure certbot for https://metrics.isample.xyz:
```
certbot --nginx -d metrics.isamples.xyz
```
8. Configure the site in the plausible.io web ui.  Remember the site name you choose as you'll need it later.
9. Create custom goals for the site (choose custom events for the goal type), corresponding to the event types enum in
`analytics.py`.
10. You should be able to test it out via curl now, and see the custom events show up on the dashboard (note that the `domain` key in the JSON corresponds to the site name in plausible):
```
curl -i -X POST https://metrics.isample.xyz/api/event \
  -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36 OPR/71.0.3770.284' \
  -H 'X-Forwarded-For: 127.0.0.1' \
  -H 'Content-Type: application/json' \
  --data '{"name":"thing_list","props":"{\"authority\":\"SMITHSONIAN\"}","url":"http://isamples.org","domain":"isamples.org"}'
```
11. Before you'll be able to see any data on the dashboard, you'll need to hit the special "pageview" event in the domain.  
    You can do so as follows:
```
curl -i -X POST https://metrics.isample.xyz/api/event \
  -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36 OPR/71.0.3770.284' \
  -H 'X-Forwarded-For: 127.0.0.1' \
  -H 'Content-Type: application/json' \
  --data '{"name":"pageview","url":"http://isamples.org","domain":"opencontext.isamples.org"}'
```
## iSamples Setup
At this point, you just need to point the iSamples web services to the deployed plausible installation.  There are two 
keys to edit in `isb_web_config.env`:
```
ANALYTICS_URL = "https://metrics.isample.xyz/api/event"
ANALYTICS_DOMAIN = "isamples.org"
```
Note that for every iSB setup, there should be a distinct plausible site, and the `ANALYTICS_DOMAIN` key should 
correspond to that sitename in plausible.

There is one file with the implementation, called `analytics.py`, but the general idea is simple.  For every API call, 
there should be a new entry in the enum, and you just call `record_analytics_event` from the api callsite with the 
corresponding `AnalyticsEvent` type.

## prometheus

[prometheus.io](https://prometheus.io/docs/introduction/overview/) is the metrics/monitoring package we have deployed for iSamples.  As of this writing, the following gauges are enabled:

* postgresql
* solr 
* node (metrics for a host instance)
* iSamples -- counts of both Things database and Solr records broken down by authority

Prometheus is deployed in a docker container, and the various bits of config are contained in the `prometheus` directory in this repository.

### miscellaneous notes about deploying on AWS
The programs that contribute the statistics (exporters, in prometheus terms), are running as part of the docker compose ensemble on iSamples in a Box.  In order for prometheus to run on a separate machine (how we have it deployed on AWS), the exporters need to bind to a port that is accessible from the machine where prometheus runs.  On AWS, this was just a matter of configuring a security group that allowed [all traffic](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-rules-reference.html#sg-rules-other-instances) with the security group, and then assigning the security group to both the iSB instance and the prometheus instance.  Note that the prometheus config *must* use the internal IP addresses within the security group to harvest the stats.  Given that [internal IP addresses stay assigned to an instance](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-ip-addressing.html) as long as it isn't decommissioned, this is acceptable for deploying long-lasting config.

### Configuring Slack alerts

Prometheus has the ability to deliver slack alerts.  We implemented this by creating a slack [webhook](https://api.slack.com/messaging/webhooks), and pointed the prometheus config (defined in `alertmanager.yml`) at the generated URL.  There are two different values for `slack_api_url` in there -- one for actual production use, and one for testing the alerts against a sandbox `#alerts-testing` slack channel.

#### Secrets

Note that Slack Webhook URLs and SMTP credentials are secrets and *should not* be checked into GitHub.  These values will need to be manually edited on the box where prometheus is deployed.

### Existing rules
There are currently (as of 5/24/2023) 3 rules: solr, postgresql, and python.  Of the 3, only the postgres exporter can do something useful if the postgres instance goes down.  Because of this, it has the simplest check:
`pg_up == 0`.  The other two checks rely on an absence of a gauge for 5 minutes using the `absent()` function.  Note that I couldn't figure out a way to write an expression that would assert that it was present for every instance.  So, I had to limit the monitored hosts to the production instance otherwise the presence of the gauge on the dev instance would cause it to not fire.  I'm sure there is a way to do it but I have given up trying for now.

### Adding new alerting rules
You can add new alerting rules by editing `prometheus_docker/prometheus.rules.yml` and adding the rules to the end of the config file.
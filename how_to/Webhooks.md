
## Webhooks install

>[!NOTE]
> Webhooks should be activated in newer Versions of NC.
> If not, enter the command below
>
> ```
>  sudo -u www-data php /var/www/nextcloud/occ app:enable webhook_listeners
> ``` 

### Create Webworkers

#### Systemd file

Create a systemd service

```
/etc/systemd/system/nextcloud-webhook-worker@.service
```

Paste this config

```
[Unit]
Description=Nextcloud Webhook worker %i
After=network.target

[Service]
ExecStart=/opt/nextcloud-webhook-worker/taskprocessing.sh %i
Restart=always
StartLimitInterval=60
StartLimitBurst=10

[Install]
WantedBy=multi-user.target
```

#### Procesor script

Now, we create the worker script called by the systemd service

```
mkdir /opt/nextcloud-webhook-worker
nano /opt/nextcloud-webhook-worker/taskprocessing.sh

```

Paste this config

```
#!/bin/sh
echo "Starting Nextcloud Webhook Worker $1"
sudo -E -u www-data php /var/www/nextcloud/occ background-job:worker -t 60 'OCA\WebhookListeners\BackgroundJobs\WebhookCall'

```

#### Starter script

This is a script to start 4 workers.

Put it somewhere you like (maybe link it to /usr/bin, so it's reachable everywhere)

```
#!/bin/bash
workers=4
echo $1
case $1 in
        start)
                for i in {1..4};
                do systemctl enable --now nextcloud-webhook-worker@$i.service;
                done;   
;;
        reload)
                for i in {1..4};
                do systemctl restart nextcloud-webhook-worker@$i.service;
                done;   
;;
        stop)
                for i in {1..4}
                do systemctl stop nextcloud-webhook-worker@$i.service
                done;
;;      
        status)
                for i in {1..4};
                do systemctl status nextcloud-webhook-worker@$i.service | grep Active:;
                done
        ;;
        list)
                systemctl list-units --type=service | grep nextcloud-webhook-worker
        ;;
        logs)
                journalctl -xeu nextcloud-webhook-worker@*.service -f
        ;;
        *)
                echo "***************************************"
                echo "***      Nextcloud Web Workers      ***"
                echo "***         managing script         ***" 
                echo "***************************************"
                echo "** Usage:                             *"
                echo "**                                    *"
                echo "** start      :start workers          *"
                echo "** stop       :stop workers           *"
                echo "** reload     :reload worker          *"
                echo "** status     :status worker          *"
                echo "** list       :list of worker         *"
                echo "** logs       :journal of workers     *"
                echo "***************************************"

        ;;
esac
```

#### Register a webhook listener







#### Troubleshoot

```
 nxtc-occ config:system:set allow_local_remote_servers --value true --type bool

```

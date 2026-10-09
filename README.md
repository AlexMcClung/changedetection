# README

## Watch for website changes and send push notifications

- Use podman and [ntfy.sh](https://docs.ntfy.sh/)
- Website to watch: [commondataset.org](https://commondataset.org/) 

### Run Locally

```
podman run -d \
  --name changedetection \
  --restart unless-stopped \
  -p 127.0.0.1:5000:5000 \
  -v changedetection-data:/datastore \
  docker.io/dgtlmoon/changedetection.io:latest
```

Then open **http://localhost:5000**. Podman creates the named volume automatically, preserving your settings and history.

Check the logs if it doesn’t load:

```
podman logs changedetection
```

Then open `http://localhost:5000`, add `https://commondataset.org/`, and configure its notification URL as:

```
ntfy://YOUR-UNIQUE-TOPIC
```

Subscribe to that same topic in your ntfy app. This integration is documented in [ntfy’s examples](<https://docs.ntfy.sh/examples/#changedetectionio>).



### Create as a service on homeserver

Assuming **Linux with Podman installed**, use a **Quadlet** to run changedetection.io and start it automatically after reboot. Quadlet is Podman’s systemd integration. [Podman documentation](<https://docs.podman.io/en/stable/markdown/podman-systemd.unit.5.html>)

**1\. Download the image and create storage**

Run these as your normal user:

```
podman pull docker.io/dgtlmoon/changedetection.io:latest
podman volume create changedetection-data
mkdir -p ~/.config/containers/systemd
```

The named volume preserves your watches, settings, and change history. The image and `/datastore` storage location come from the [official container instructions](<https://github.com/dgtlmoon/changedetection.io>).

**2\. Create the service configuration**

```
cat > ~/.config/containers/systemd/changedetection.container <<'EOF'
[Unit]
Description=Website change monitor

[Container]
Image=docker.io/dgtlmoon/changedetection.io:latest
ContainerName=changedetection
PublishPort=127.0.0.1:5000:5000
Volume=changedetection-data:/datastore

[Service]
Restart=always
TimeoutStartSec=300

[Install]
WantedBy=default.target
EOF
```

**3\. Start it and enable running while logged out**

```
systemctl --user daemon-reload
systemctl --user start changedetection.service
sudo loginctl enable-linger "$USER"
```

The `[Install]` section handles automatic startup; you don’t need to run `systemctl enable` for the generated Quadlet service.

Check its status:

```
systemctl --user status changedetection.service
```

**4\. Configure the website and ntfy**

Open **http://localhost:5000** on the computer running Podman.

- Add `https://commondataset.org/` as a watch.
- Set its check interval to **1 day**.
- Under **Edit → Notifications**, add:

  ```
  ntfy://YOUR-UNIQUE-TOPIC
  ```
- Subscribe to that topic in your phone’s ntfy app using server `https://ntfy.sh`.
- Send a test notification from changedetection.io.

This notification URL format is documented in [ntfy’s changedetection.io integration guide](<https://docs.ntfy.sh/examples/#changedetectionio>).

If Podman runs on a remote server, open an SSH tunnel from your computer:

```
ssh -L 5000:127.0.0.1:5000 your-user@your-server
```

Then visit **http://localhost:5000** locally.

For troubleshooting, view the service logs:

```
journalctl --user -u changedetection.service -n 50 --no-pager
```

The `changedetection-data` volume remains available for the Quadlet to reuse.

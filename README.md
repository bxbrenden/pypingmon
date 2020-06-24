# pypingmon
`pypingmon` is a python-based ping monitoring solution for your server.
It periodically sends a number of ping packets (ICMP echo requests) to your server, and if 100% of them fail, it sends you an SMS message via Twilio.
Once your server is detected as down, `pypingmon` continues pinging to detect when your server comes back up.
Once the server is back up, `pypingmon` automatically starts monitoring it again and sends an SMS alert if it goes down again.
You can configure the number of ping packets `pypingmon` tries to send before alerting you.

## Requirements
- `python` >= 3.6
- A [Twilio](https://www.twilio.com/) account with a phone number and  enough balance to send SMS messages.
- root access to a Linux server with `systemd` (only if running as a service)

## Configuration
`pypingmon` sets all unique variables through environment variables.
The required environment variables for the monitoring service are:
| Environment Variable | Description | Example |
|----------------------|-------------|---------|
| `PYPINGMON_WATCH_HOST`| the name of the server you want to monitor | example.com |
| `PYPINGMON_NUM_PINGS` | the number of pings to attempt before alerting | 30 |
| `TWILIO_SID` | your Twilio SID (account number) | ABCDEFGHIJKL01234567|
| `TWILIO_TOKEN` | your Twilio API token | ZYXWVUTSRQPONM987654321 |
| `TWILIO_SRC_NUM` | the Twilio phone number to text from | +15034206969 |
| `TWILIO_DST_NUM` | the phone number to send the text alert to | +13035552600 |

## Usage
`pypingmon` currently runs directly as a command-line script or as a systemd service.
This section explains how to run it as a script from the command line.

Install the required modules with `pip` (in some cases `pip3`) as follows:

```bash
pip install -r requirements.txt
```

To invoke `pypingmon`, set all environment variables as noted above, and then run:

```bash
chomd +x pypingmon
./pypingmon
```

pypingmon will continuously ping your host.
Once the host is no longer pingable, `pypingmon` texts your destination phone number exactly once with an alert message.

## Example Output
As long as your host is pingable, `pypingmon` displays the same repetitive message:

```
Pinging example.com was successful
Pinging example.com was successful
Pinging example.com was successful
```

When the ping request completely fails, `pypingmon` displays this message:
```
Pinging example.com has failed 1 time
Pinging example.com has failed 2 times
Pinging example.com has failed 3 times
```

When ping failures cross the threshold, `pypingmon` sends a text message like this:

```
ALERT: server "example.com" failed ping test 3 times
```

When ping failures cross the threshold, the following message appears in the `pypingmon` logging output, and the service stops running:

```
Server example.com was detected as down. Closing to prevent unlimited text notifications'
```

## Running as a Systemd service
To run pypingmon as a `systemd` service, you need to manually configure a few things.

First you'll need to create a symbolic link of `pypingmon` to a directory that's in your $PATH and is executable.

For example, the directory `/usr/local/bin` is in my $PATH, so I use these commands:

```bash
chmod a+x pypingmon
sudo ln -s `pwd`/pypingmon /usr/local/bin/pypingmon
```

Next you'll need to copy the `pypingmon.service` file from the root of the project into `/etc/systemd/system/`.
Ensure that `/etc/systemd/system/pypingmon.service` is owned by root and has the proper permissions with the following commands:

```bash
sudo cp pypingmon.service /etc/systemd/system/
sudo chown root:root /etc/systemd/system/pypingmon.service
sudo chmod 644 /etc/systemd/system/pypingmon.service
```

Next you will need to create a `systemd` Environment File that contains the environment variables used by `pypingmon`.

Create the directory `/etc/systemd/system/pypingmon.service.d`, copy the `pypingmon.conf.example` file from this directory into it as `gpypingmon.conf`, and configure the file with your values.

```bash
sudo mkdir /etc/systemd/system/pypingmon.service.d
sudo cp pypingmon.conf.example /etc/systemd/system/pypingmon.service.d/pypingmon.conf
sudo chown root:root /etc/systemd/system/pypingmon.service.d/pypingmon.conf
sudo chmod 644 /etc/systemd/system/pypingmon.service.d/pypingmon.conf
```

Next, run the following commands to register the `systemd` service, start it, and check its status:

```bash
sudo systemctl daemon-reload
sudo systemctl start pypingmon.service
sudo systemctl status pypingmon.service
```

If all goes well, you should see output similar to the following:

```
● pypingmon.service - pypingmon ping monitoring and SMS alerting service
   Loaded: loaded (/etc/systemd/system/pypingmon.service; disabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/pypingmon.service.d
           └─pypingmon_env.conf
   Active: active (running) since Mon 2020-05-25 00:09:39 PDT; 10min ago
     Docs: https://github.com/bxbrenden/pypingmon/blob/master/README.md
 Main PID: 1721 (python3)
    Tasks: 2 (limit: 4915)
   Memory: 22.6M
   CGroup: /system.slice/pypingmon.service
           ├─1721 python3 /usr/local/bin/pypingmon
           └─2247 ping -c 10 example.com
```

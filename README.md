# fly-vsftpd

A simple FTP server for [Fly.io](https://fly.io) (based on [fly-ftp-server](https://github.com/gregmsanderson/fly-ftp-server)) using `vsftpd`. README is slightly out of date, but follow the commands and it should work.

## Customise

Edit the `name` in the `fly.toml` to one of your choice:

```toml
app = "fly-vsftpd"
```

You may also want to adjust the FTP options. Take a look at the `conf/vsftpd.conf` file, adjusting that to your needs. We have generally used default values.

## IP
This needs a dedicated ipv4 address allocated to your app.

```
fly ips allocate-v4
```

## Deploy

**Note:** Answer _NO_ at the end, since you need to do a couple of things before you can deploy:

```
$ fly launch
An existing fly.toml file was found
? Would you like to copy its configuration to the new app? Yes
Creating app in /path/to/it
Scanning source code
Detected a Dockerfile app
? App Name (leave blank to use an auto-generated name): your-name-here
? Select organization: personal (personal)
? Select region: lhr (London)
Created app your-name-here in organization personal
Wrote config file fly.toml
? Would you like to setup a Postgresql database now? No
? Would you like to deploy now? No
Your app is ready. Deploy with `flyctl deploy`
```

Why not deploy right now?

1. You need to create a volume to store the uploaded files.

2. You need to specify the `USERS` that are permitted to connect to the server.

So let's do that:

## Storage

The provided storage is ephemeral so you will need to create a [volume](https://fly.io/docs/reference/volumes/) in the region you plan on deploying your app in (for example `lhr`). The size is in GB. For example:

```
fly volumes create ftp_data --region <REGION> --size 1
```

Volumes are by default encrypted. The command should return its details to show it was successful.

## Users

The `USERS` value must be a space-separated list of users. Each user _must_ have a name and password. And can _optionally_ have a home folder. And _optionally_ a UID. We'll mount the volume above to the `/data` folder. So for example you _could_ run this which would create a user called **example**, with a password of **example**:

```
fly secrets set USERS="example|example|/data/example"
```

Obviously your command should use a better name and a more secure password. Each user should be separated by a space. For example these examples create _two_ users:

```
USERS="username|password|/data/username foo|bar|/data/foo"
USERS="username|password|/data/username|5000 foo|bar|/data/foo|5001"
```

Run that and you should then see:

```
Secrets are staged for the first deployment
```

Now you can go ahead and deploy the app:

```
fly deploy
```

You should see it build, push the image, then create the release. There will be some warnings about the passive ports, but you can ignore them.

Confirm the app is running with `fly status`. Now try connecting to it using your FTP client. Use `your-app-name.fly.dev`/`<DEDICATED IP>` as the hostname, 21 as the port, and one of the username/password from your secret `USERS` value. You will need to use plain FTP and there will probably be a warning about security.

## Debugging

Use `fly logs` to see what is output and look for any errors. If you have left our default TCP healthcheck in the `fly.toml`, you should see _its_ connections roughly every five seconds:

```
2022-06-08T16:02:09Z app[abcdefg] lhr [info]Wed Jun  8 16:02:09 2022 [pid 2] CONNECT: Client "1.2.3.4"
2022-06-08T16:02:14Z app[abcdefg] lhr [info]Wed Jun  8 16:02:14 2022 [pid 2] CONNECT: Client "1.2.3.4"
```

Try running `fly ssh console` to connect to a vm. Once there you can check `vsftpd` is running e.g:

```
ps -a | grep vsftpd
528 root      0:00 pidproxy /var/run/vsftpd/vsftpd.pid true
550 root      0:00 vsftpd /etc/vsftpd/vsftpd.conf
722 root      0:00 grep vsftpd
```

From there you can also check files uploaded by users (into the `/data` folder). Run `cd data` and you should see the folders for your users e.g `/data/username`. Check the folder's owner matches the one _you_ set in your `USERS` secret. If not, that would explain if a particular user can't connect. And look inside the folder to see the files are present and valid.

This basic FTP server does not use FTPS. In theory you should be able to modify it to use Fly's provided TLS handler or provide your own certificate.

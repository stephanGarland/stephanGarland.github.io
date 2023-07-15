---
title: 'Self-hosting a forum with Discourse'
date: Tue, 14 Jul 2020 22:09:26 +0000
draft: false
tags: ['#self-hosting', '#socialmedia', '#docker']
---

What do you do if you want a private social media experience for you and friends or family? Facebook has Groups, and lesser-known apps like Band exist that can also fill that niche, but you're still handing over all your content to a 3rd party. For some, that's untenable, and completely understandable. Furthermore, I wouldn't have anything of interest to write if the answer was "Make a Group. Now set it to Private. Invite people."

I looked at various open-source forum offerings, and eventually settled on [Discourse](https://www.discourse.org/). It's modern-ish, is Docker-native, isn't written in PHP, and has plugins for days. Administering it definitely still has a hacky feel, but I assume if you're reading this blog you have the ability to handle it.

I'll defer to [their own guide](https://github.com/discourse/discourse/blob/master/docs/INSTALL-cloud.md) for the actual install, and simply add in steps necessary to set it up how I did, which is:

1.  Self-hosting base Docker image on own server behind an nginx reverse proxy
2.  Uploads and backups directed to an encrypted S3 bucket with no public access
1.  Optional: Add a CDN via Cloudfront
3.  DNS handled by Cloudflare
4.  TLS handled by Let's Encrypt
5.  Mail provider via [Mailjet](https://www.mailjet.com/)

You're free to switch things out as you see fit. I chose S3 because it's natively supported, and I'm already familiar with AWS. I chose Cloudflare because I already use it, and it works quite well. I chose Mailjet because they offer a free tier with no expiry and no credit card needed for signup.

### Domain name

You'll need a domain name. Buy one from wherever you want. If you want to have resolution handled by Cloudflare, like I'm doing, make an account with them and switch the nameservers over.

### Mail server

You need a mail server to set up Discourse. They have huge warnings about this in the setup guide. You'll also need to set up a SPF and DKIM record. I'll [defer to Mailjet](https://app.mailjet.com/docs/spf-dkim-guide) for that; note that you may also need a separate TXT record to verify your domain; your mail server will inform you if it's needed.

### Nginx setup

I host multiple Docker apps on my server and my RPi 4, and access them via nginx running on the server. If you don't need this, you can skip this, and just run the normal setup method described in Discourse's install guide. Otherwise, read on.

1.  Install Nginx by whatever means you'd like, natively or within Docker. If native, set it up as a service.
2.  In `/etc/nginx/nginx.conf`, add the following within either the http{} block (affects all sites) or a specific server{} block:
1.  `client_max_body_size 16M;`
2.  `# I selected 16 MB here, but you can go with whatever you think is a reasonable maximum size for uploads`
3.  In `/etc/nginx/sites-available/$YOUR\_SITE\_URL.conf`:

```
# Note that I'm not using a socket here, as Discourse recommends.
# This is because I already have everything else using ports, so consistency.
# If you want to use a socket, it would look like this:
# proxy_pass http://unix:/var/discourse/shared/standalone/nginx.http.sock:;
server {
server_name $YOUR_URL;
location / {
proxy_pass http://$YOUR_SERVER_IP:$YOUR_PORT;
proxy_set_header Host $http_host;
proxy_http_version 1.1;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Real-IP $remote_addr;
}
}
```

4.  `ln -s /etc/nginx/sites-available/$YOUR_SITE_URL.conf /etc/nginx/sites-enabled/$YOUR_SITE_URL.conf`
5.  `sudo systemctl reload nginx`

### Let's Encrypt setup

I'm using [Certbot](https://certbot.eff.org/) to handle my certificates.

1.  Install certbot by whatever means you'd like.
2.  `sudo certbot --nginx`
3.  Follow any instructions given, and test that it worked.

### DNS records

As mentioned, you'll need an SPF and DKIM record, and potentially a TXT record to verify your domain for the mail server. Set them up with your DNS provider as given by your mail provider, and don't forget that it can take some time (for me, ~5-10 minutes with Mailjet and Cloudflare) for the changes to be seen.

### Discourse setup

Since you're running an nginx reverse proxy, you can't use their setup script, which assumes port 80 is open.

1.  `cp /var/run/discourse/samples.yml /var/discourse/containers/$DESIRED_DOCKER_CONTAINER_NAME.yml`
2.  Open this new file in the editor of your choice.
3.  Comment out `templates/web.ssl.template.yml` and `templates/web.letsencrypt.ssl.template.yml` under `templates`.
1.  If you wanted to use sockets instead of ports, add `templates/web.socketed.template.yml`, and comment out the entire `expose` section.
4.  Comment out `443` under `expose`, and replace the mapping to `80` to the port you chose earlier, e.g. `- "8888:80"`
5.  Add your site's URL to `DISCOURSE_HOSTNAME`.
6.  Add one or more emails to `DISCOURSE_DEVELOPER_EMAILS` - these will be site admins.
7.  Add SMTP information to `DISCOURSE_SMTP_*`
1.  For Mailjet, the API ID and Key are the Username and Password in this file, respectively.
2.  Yes, these are stored in plaintext. If you want to integrate Vault or something into this, please write it up, I'd be thrilled to see it.
8.  `sudo ./launcher bootstrap $DESIRED_DOCKER_CONTAINER_NAME`
1.  This will take a fair amount of time, and when it's done, you should have a Docker container running - if everything else is set up correctly, you should be able to hit the URL from your browser.
2.  You must verify that HTTPS is working at this point. It is likely that you'll get an Info warning, saying that you have a valid certificate, but that some elements are being delivered insecurely. This will be rectified later. If instead it's completely insecure, something is wrong, and it needs to be fixed before continuing.
9.  Follow the instructions on your Discourse instance to verify an admin email. If you've not yet set up your DNS records for mail, this will probably fail, and you'll get an email from your mail provider nagging you to fix the problem.

### S3 setup

1.  Create an S3 bucket, with all public access blocked, and encryption at rest.
1.  According to forum posts, you should be able to skip this step if the IAM is set up correctly, as Discourse will create the necessary buckets. YMMV.
2.  Create an IAM policy. The example is here, if you'd like to copy it.
1.  It could be better organized to be fair, as the backup doesn't actually need all of those permissions. Since these were limited in scope and closed to the public, I didn't care to further limit them, but you may.
2.  Lifecycling can be done from within Discourse, hence the inclusion of `PutLifecycleConfiguration` - if you'd rather handle that from S3, you can remove it.
3.  Similarly, if you manually created the buckets, you can remove `CreateBucket`.

```
{
"Version": "2012-10-17",
"Statement": [
{
"Effect": "Allow",
"Action": [
"s3:List*",
"s3:Get*",
"s3:AbortMultipartUpload",
"s3:DeleteObject",
"s3:PutObject",
"s3:PutObjectAcl",
"s3:PutObjectVersionAcl",
"s3:PutLifecycleConfiguration",
"s3:CreateBucket",
"s3:PutBucketCORS"
],
"Resource": [
"arn:aws:s3:::$BACKUP_BUCKET_NAME",
"arn:aws:s3:::$BACKUP_BUCKET_NAME/*",
"arn:aws:s3:::$UPLOAD_BUCKET_NAME",
"arn:aws:s3:::$UPLOAD_BUCKET_NAME/*"
]
}
]
}
```

3.  Create an IAM user, and assign this policy to them. Copy the Access ID and Secret Key, and put them somewhere safe, ideally something like Lastpass or 1Pass. You'll need them for Discourse.

### Discourse configuration

1.  In Settings --> Login:
1.  Select `login required` and `must approve users`.
2.  You could additionally add an `invite code`.
3.  If you plan on setting up OAuth via Google, Facebook, et. al., this is where you'll configure them.
2.  In Settings --> Security:
1.  Select `force https`.
3.  In Settings --> Files:
1.  Change `max image size kb` and `max attachment size kb` to match the value you selected for nginx.
2.  Enter your S3 credentials and upload bucket name into their respective fields, and select the proper region for your bucket.
1.  If you were using Cloudfront, I assume you've set it up, including an OAI, and granted it permission in the S3 Bucket Policy.
2.  You'd then enter its endpoint into `s3 cdn url`.
3.  If you have issues with this, you can contact me, as I did get it working - I just decided it wasn't necessary for my use case.
3.  Select `enable s3 uploads`.
4.  Select `prevent anons from downloading files`, and`secure media`.
4.  In Settings --> Backups:
1.  Enter your S3 bucket name into `s3 backup bucket`.
2.  Change `backup location` to `s3`.
3.  Change `maximum backups` if you'd like Discourse to handle lifecycling for you.
4.  Enable `automatic backups enabled` unless you'd rather manually run backups.
5.  Select `include thumbnails in backups` if desired - it makes for a faster restore should it be needed.
6.  Change `backup frequency` and `backup time of day` to whatever makes sense for you - I'm using daily (`1`), and `05:00` UTC.

### Testing it out

1.  In Backups (not Settings --> Backups), generate a backup, and verify that you have a tarball in the s3 bucket.
2.  Create a new topic, and try both text and image posts.
1.  For an image, if you have S3 permissions issues, it won't upload at all, and you'll get an `Access Denied` error.
2.  You should be able to view the image from within Discourse, and only from within Discourse - even viewing the file directly from the S3 bucket should fail.
3.  Try manually accessing a URL to an image from within Discourse in an Incognito window. If you include the signed part after the path, it should work. If you don't, it should fail.
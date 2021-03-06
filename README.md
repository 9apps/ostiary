# Ostiary (ancient bouncer)

Wherever exclusivity is required, it has to be enforced. Evidently, this also holds for the digital realm. Traditional approaches to (central) user management are cumbersome and/or expensive. And they certainly are not ready for the new world of 'the cloud'.

AWS has IAM, for policies on AWS' services and assets. They also offer keypairs, which are given to launching instances, and give anonymous access to instances. We require a bit more personalized implementation of keypairs. (Paper trail and all that...)

## Install aws ami (centos)

    curl https://raw.githubusercontent.com/9apps/ostiary/master/users > /etc/init.d/users
    chkconfig --add users
    chkconfig --level 23456 users on
    chkconfig --level 01 users off

## Install on ubuntu

    curl https://raw.github.com/9apps/ostiary/master/users_ubuntu > /etc/init.d/users_ubuntu
    chmod 755 /etc/init.d/users_ubuntu
    update-rc.d users_ubuntu start 01 1 2 3 4 5 6 . stop 98 0 .

This is it. All that is required now is a bucket of keys, augmented userdata, and the privilege to get the key objects from the bucket. (We often use IAM roles for that.)

## Userdata

Launching an instance with the following as userdata will create users

    {
        "persistence": {
            "type": "raid",
            "count" : 2,
            "size": "10"
        },
        "backup" : {
            "prefix" : "logging-30mhz-com",
            "bucket" : "elasticsearch.30mhz.com"
        },
        "security" : {
            "bucket" : "keys.30mhz.com",
            "users" : {
                "jasper" : { "groups" : [ "wheel" ], "email" : "jasper@9apps.net", "comment" : "Jasper Geurtsen" },
                "flavia" : { "groups" : [ "wheel" ], "email" : "flavia@9apps.net", "comment" : "Flavia Paganelli" },
                "pimderneden" : { "groups" : [ "wheel" ], "email" : "pim@9apps.net", "comment" : "Pim Derneden" },
                "jurg" : { "groups" : [ "wheel" ], "email" : "jurg@9apps.net", "comment" : "Jurg van Vliet" }
            }
        },
        "monit": {
            "to": "30mhz@9apps.net",
            "domain": "30mhz.com",
            "system": "logging.elasticsearch.30mhz.com"
        }
    }

This userdata points to the bucket keys.30mhz.com, and expects an object with the name `jasper@9apps.net`, for the key of the user `jasper`.

NOTE: for ubuntu, change in the userdata: <"groups" : [ "wheel" ]> into <"groups" : [ "sudo" ]>
as Ubuntu has no wheel group!

## Requirements

We have three different use-cases
1. adding a user
2. updating a user
3. deleting/disabling a user

Adding a user is usually not a matter of great rush. It is, however, a cumbersome exercise to add a user to all different AMIs in a large environment. It might take several hours to update AMIs, test them, and rotate running instances. So, adding users can be a bit cumbersome, but should not require creating new AMIs.

	{
	  "Version": "2012-10-17",
	  "Statement": [
		{
		  "Action": [
			"s3:GetObject",
			"s3:GetObjectTorrent",
			"s3:GetObjectVersion",
			"s3:GetObjectVersionTorrent"
		  ],
		  "Resource": [
			"arn:aws:s3:::keys.30mhz.com/*"
		  ],
		  "Effect": "Allow"
		}
	  ]
	}

Updating and deleting/disabling a user is often sensitive. If someone is fired, or leaves disgruntled, you would want to deny access as soon as possible. And, if a laptop is stolen, changing public keys is also something you would want to do as soon as possible.

User accounts on instances in most systems are only used for (administering) access. Keeping personal files for long times is not required. The only things we need to know about a user is
* username
* email
* full name (comment)
* groups
* authorized keys

We put username, email, full name and groups in userdata. Authorized keys are stored in S3, in objects with the email as their key name.

---
layout: post
title: The PWA Experiment - pt II
subtitle: Testing and debugging on device
---

To convert a website into a progressive web app, a couple of things are required:

- the website needs to have a valid manifest file
- the website needs to have a registered service worker
- the website needs to be served over SSL with a valid certificate

In order to properly test and debug your PWA, SSL is required as well. Some service worker bits will accept localhost as a trusted source but a lot of functionality won't work at all without a proper https setup.

# Setup SSL

For local debugging, we're going to use a self-signed certificate. This certificate will be linked to the 'localhost' domain and only be trusted by my dev machine.

To create the self-signed certificate I've setup a gist with the proper commands and files necessary: [https://gist.github.com/msioen/4f440ec49eec3254941ce3e4c6ff7982](https://gist.github.com/msioen/4f440ec49eec3254941ce3e4c6ff7982). As this certificate will be used for local testing we're running everything on localhost. All the defaults can stay as-is and you only need to execute the *create_server_cert_and_key* script. This will create the server.crt and server.key files which you'll need to setup SSL.

_Note: if you'll need the certificate to work on an Android device you may want to read the next section before creating these files_

The certificate we just created is self-signed and won't be trusted by any system or browser as it's not signed by an authorized authority. For local debugging we can work around this by force trusting this certificate. Double click on the certificate to import it in the keychain. After the cert is imported you can change it' trust settings to 'Always trust'.

![](/img/pwa-setup-trust.png "trust the certificate")

The last necessary change is updating the manifest file of your website. This is required to ensure your website will be treated as a progressive web app and not just as a normal website. One of the required manifest properties is called start_url. For local debugging this needs to match the actual url you will be going to, for production this should be the actual domain name. In my case I switch it to https://localhost:4000/ for local debugging and https://michielsioen.be/ for production bits.

With the above setup steps finished we're ready to test the SSL setup in a browser. I'm using Jekyll to host my site. In order to test locally with the created certificate you need to move the .crt and .key files to the root of your website and use the following command to use them.

```
bundle exec jekyll serve --host=0.0.0.0 --ssl-cert server.crt --ssl-key server.key
```

_Note: the --host=0.0.0.0 is not required at this point. As it's required to test on a mobile phone it's included here for completeness._

When browsing to localhost, the website should now be indicated as having a valid certificate. On recent Chrome versions you should now get the option to install your PWA as well. If this isn't the case, double check the requirements listed here: [https://developers.google.com/web/fundamentals/app-install-banners/#criteria](https://developers.google.com/web/fundamentals/app-install-banners/#criteria)

![](/img/pwa-setup-secured.png "accepted https connection")

# Device

Our PWA can be served over SSL now. If testing on desktop with the Google Chrome browser is good enough, you can stop here. If everything needs to work on mobile as well, several other changes are required.

The first issue we need to solve is that our self-signed certificate is linked to the localhost domain. As we're not running a webserver on our mobile phone, browsing to localhost won't work yet. At this point you have to browse to the IP address of your computer.

To resolve this, we can setup port forwarding via Chrome developer tools. The official documentation can be found on the google developer pages which explains nicely what to do: [https://developers.google.com/web/tools/chrome-devtools/remote-debugging/local-server](https://developers.google.com/web/tools/chrome-devtools/remote-debugging/local-server). When it's setup and your device is linked the 'Remote devices' tab of the Chrome developer tools should look somewhat like this:

![](/img/chrome-port-forwarding.png "chrome developer tools - port forwarding")

We're now able to browse to localhost on our phone with the port setup in chrome.

<img src="/img/android-localhost-insecure.png" title="port forward setup - site still insecure" style="max-width: 50%; display: block; margin-right: auto; margin-left: auto;" />

Secondly, we need to get our phone to trust the self-signed certificate we created. Because of some changes Android made to how they deal with certificates, it's not possible to trust the certificate we made previously. Instead we need to make two new certificates: a self-signed certificate authority and a server certificate signed by this authority. Once we get Android to trust our own certificate authority, everything signed by it will be trusted.

I've prepared a gist with everything necessary to create these files: [https://gist.github.com/msioen/1cf224a242249c223cc7eee2ded9240d](https://gist.github.com/msioen/1cf224a242249c223cc7eee2ded9240d). There are two separate scripts to execute here, *create_root_cert_and_key* to create all necessary certificate authority bits and *create_server_cert_and_key_with_ca* to create our server files. As before, for localhost development all defaults should be ok here.

The file that needs to be installed on your phone to get the website trusted is rootCA.crt. After transferring this file on your phone, you should be able to click on it which will start installation. Android should say that this certificate contains CA certificates and not user certificates. If it mentions that you're installing a user certificate something went wrong in one of the earlier steps.

<img src="/img/android-install-ca.png" title="install certificate authority on Android" style="max-width: 50%; display: block; margin-right: auto; margin-left: auto;" />

At this point you should be able to browse to your PWA on device without security issues. The port forwarding setup ensures we browse to localhost, the trusted certificate authority ensures our self-signed certificate is considered secure.

<img src="/img/pwa-device-secured.png" title="accepted https connection" style="max-width: 50%; display: block; margin-right: auto; margin-left: auto;" />

_Note: if you also update the manifest.json start_url with the port used in Chrome port forwarding, Android Chrome will automatically suggest to install your site at this point._

# Debug

The major setup work has been done in the previous sections. In order to debug on device we first need to be able to run the site on the phone as described in the previous section. At this point you're able to hook the Chrome developer tools into the website on the phone.

The 'Remote devices' tab in the developer tools shows the last opened tabs on device with an 'Inspect' button next to it. If you installed your PWA on device and you launch it, it will show up in this list. Alternatively, you can open it in the browser and hook into a non-installed version.

Once DevTools has opened this way you can debug the way you can with normal websites. Add breakpoints, inspect console messages, measure performance, ... All the things you'd expect.

![](/img/pwa-devtools.png "chrome developer tools - debug pwa on phone")


<br />
<br />

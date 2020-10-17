---
layout: post
title: Mutual SSL authentication on .NET Core
subtitle: Tracking down issues on Linux
---

Mutual SSL authentication with .NET Core is fairly straightforward. This week I ran into some interesting (at the time very frustrating) exceptions however, when our application was deployed on the target Linux machine. This post is mainly intended to possibly help inform other developers running into the same exceptions.

I'll briefly explain what SSL authentication is before going into details about what went wrong in our case.

# Mutual SSL authentication on .NET Core 3.1

Mutual authentication is the concept of two parties authenticating each other simultaneously. Most of time this is applied for backend to backend authentication as a control to manage who is authorized to access your server without user logins. In practice this means both client and server have a X.509 certificate. During the SSL handshake between client and server, both parties present their certificate. If one side can't verify the certificate with its private key the authentication fails.

```csharp
    private static async Task ConnectToServer()
    {
        // load in the certificate
        // !! this needs to have both public and private key part in order to work
        var pfxCertificateBase64 = "{YOUR_PFX_IN_BASE64_HERE}";
        var pfxPassword = "{YOUR_PFX_PASSWORD_HERE}";
        var clientCertificate = ConvertFromPfx(pfxCertificateBase64, pfxPassword);

        // prepare our httpclient handler, pass in our certificate
        var clientHandler = new HttpClientHandler();
        clientHandler.ClientCertificates.Add(clientCertificate);
        clientHandler.ClientCertificateOptions = ClientCertificateOption.Manual;

        // prepare our httpclient
        // !! in production code you want to reuse the same client for all calls
        var httpClient = new HttpClient(clientHandler)
        {
            BaseAddress = new Uri("https://localhost:58788")
        };

        // fetch something from our endpoint
        await httpClient.GetAsync("/api/v1/init");
    }

    private static X509Certificate2 ConvertFromPfx(string pfx, string password)
    {
        var certificate = Convert.FromBase64String(pfx);

        var clientCertificate = new X509Certificate2(certificate, password,
            X509KeyStorageFlags.Exportable);

        return clientCertificate;
    }
```

It's also worth it to maybe give the briefest overview of how certificates are created. In our digital world we have the concept of certificate authorities. These are entities which issue digital certificates and certify them to be valid / maintain lists of invalid certificates. Certificate authorities manage root certificates which in turn are used to generate actual certificates (often with one or more intermediates in between). This means that every digital certificate which is created links back to a root certificate from a certificate authority.

In order to verify/validate a certificate, the certificate itself is validated, the full chain up to the root certificate is validated and the system checks if it actually trusts this root certificate or not. Only if the root is trusted and all certificates properly link to each other, the certificate will be deemed valid.

Every computer has a list of 'approved' root certificates by default. Without these you would not be able to do anything on the internet. Aside from the default approved commercial and non-commercial root certificates you can also manually trust your own or self-signed root certificates.

In our use case the client certificate was signed with a custom certificate authority with all correct fields in place and accessible to the internet. The certificate chain had 3 certificates: root ca - intermediate - client.


For more extensive documentation, [Microsoft docs](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/certauth?view=aspnetcore-3.1) has a great overview of the full setup for both client and server side.

# Works on my machine - ship it

The code to properly authenticate against our server is written. It's tested locally and works. However, after deploying (in our case to an Azure Linux App Service) the application consistently crashes. This always after a freeze of 1 minute and 40 seconds without ever getting to the server. Following stacktrace could be observed.

```
System.Threading.Tasks.TaskCanceledException: A task was canceled.
   at System.Net.Security.SslStream.StartSendBlob(Byte[] incoming, Int32 count, AsyncProtocolRequest asyncRequest)
   at System.Net.Security.SslStream.ForceAuthentication(Boolean receiveFirst, Byte[] buffer, AsyncProtocolRequest asyncRequest)
   at System.Net.Security.SslStream.ProcessAuthentication(LazyAsyncResult lazyResult, CancellationToken cancellationToken)
   at System.Net.Security.SslStream.BeginAuthenticateAsClient(SslClientAuthenticationOptions sslClientAuthenticationOptions, CancellationToken cancellationToken, AsyncCallback asyncCallback, Object asyncState)
   at System.Net.Security.SslStream.<>c.<AuthenticateAsClientAsync>b__65_0(SslClientAuthenticationOptions arg1, CancellationToken arg2, AsyncCallback callback, Object state)
   at System.Threading.Tasks.TaskFactory`1.FromAsyncImpl[TArg1,TArg2](Func`5 beginMethod, Func`2 endFunction, Action`1 endAction, TArg1 arg1, TArg2 arg2, Object state, TaskCreationOptions creationOptions)
   at System.Threading.Tasks.TaskFactory.FromAsync[TArg1,TArg2](Func`5 beginMethod, Action`1 endMethod, TArg1 arg1, TArg2 arg2, Object state, TaskCreationOptions creationOptions)
   at System.Threading.Tasks.TaskFactory.FromAsync[TArg1,TArg2](Func`5 beginMethod, Action`1 endMethod, TArg1 arg1, TArg2 arg2, Object state)
   at System.Net.Security.SslStream.AuthenticateAsClientAsync(SslClientAuthenticationOptions sslClientAuthenticationOptions, CancellationToken cancellationToken)
   at System.Net.Http.ConnectHelper.EstablishSslConnectionAsyncCore(Stream stream, SslClientAuthenticationOptions sslOptions, CancellationToken cancellationToken)
   at System.Net.Http.HttpConnectionPool.ConnectAsync(HttpRequestMessage request, Boolean allowHttp2, CancellationToken cancellationToken)
   at System.Net.Http.HttpConnectionPool.CreateHttp11ConnectionAsync(HttpRequestMessage request, CancellationToken cancellationToken)
   at System.Net.Http.HttpConnectionPool.GetHttpConnectionAsync(HttpRequestMessage request, CancellationToken cancellationToken)
   at System.Net.Http.HttpConnectionPool.SendWithRetryAsync(HttpRequestMessage request, Boolean doRequestAuth, CancellationToken cancellationToken)
   at System.Net.Http.RedirectHandler.SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
   at System.Net.Http.DiagnosticsHandler.SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
   at MyAppNamespace.LoggingHandler.SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
```

Interestingly enough, if the HttpClient.Timeout property is manually set, you get a different exception. Following stacktrace occurred with the exact same code as above, with the only difference being: HttpClient.Timeout = TimeSpan.FromMinutes(3). Just as before, this exception throws after 100 seconds, without ever hitting the server. 

```
System.Net.Http.HttpRequestException: The SSL connection could not be established, see inner exception.
   ---> System.IO.IOException: Authentication failed because the remote party has closed the transport stream.
   at System.Net.Security.SslStream.StartReadFrame(Byte[] buffer, Int32 readBytes, AsyncProtocolRequest asyncRequest)
   at System.Net.Security.SslStream.StartReceiveBlob(Byte[] buffer, AsyncProtocolRequest asyncRequest)
   at System.Net.Security.SslStream.CheckCompletionBeforeNextReceive(ProtocolToken message, AsyncProtocolRequest asyncRequest)
   at System.Net.Security.SslStream.StartSendBlob(Byte[] incoming, Int32 count, AsyncProtocolRequest asyncRequest)
   at System.Net.Security.SslStream.ForceAuthentication(Boolean receiveFirst, Byte[] buffer, AsyncProtocolRequest asyncRequest)
   at System.Net.Security.SslStream.ProcessAuthentication(LazyAsyncResult lazyResult, CancellationToken cancellationToken)
   at System.Net.Security.SslStream.BeginAuthenticateAsClient(SslClientAuthenticationOptions sslClientAuthenticationOptions, CancellationToken   cancellationToken, AsyncCallback asyncCallback, Object asyncState)
   at System.Net.Security.SslStream.<>c.<AuthenticateAsClientAsync>b__65_0(SslClientAuthenticationOptions arg1, CancellationToken arg2,   AsyncCallback callback, Object state)
   at System.Threading.Tasks.TaskFactory`1.FromAsyncImpl[TArg1,TArg2](Func`5 beginMethod, Func`2 endFunction, Action`1 endAction, TArg1 arg1,   TArg2 arg2, Object state, TaskCreationOptions creationOptions)
   at System.Threading.Tasks.TaskFactory.FromAsync[TArg1,TArg2](Func`5 beginMethod, Action`1 endMethod, TArg1 arg1, TArg2 arg2, Object state,   TaskCreationOptions creationOptions)
   at System.Threading.Tasks.TaskFactory.FromAsync[TArg1,TArg2](Func`5 beginMethod, Action`1 endMethod, TArg1 arg1, TArg2 arg2, Object state)
   at System.Net.Security.SslStream.AuthenticateAsClientAsync(SslClientAuthenticationOptions sslClientAuthenticationOptions, CancellationToken   cancellationToken)
   at System.Net.Http.ConnectHelper.EstablishSslConnectionAsyncCore(Stream stream, SslClientAuthenticationOptions sslOptions, CancellationToken   cancellationToken)
   --- End of inner exception stack trace ---
   at System.Net.Http.ConnectHelper.EstablishSslConnectionAsyncCore(Stream stream, SslClientAuthenticationOptions sslOptions, CancellationToken   cancellationToken)
   at System.Net.Http.HttpConnectionPool.ConnectAsync(HttpRequestMessage request, Boolean allowHttp2, CancellationToken cancellationToken)
   at System.Net.Http.HttpConnectionPool.CreateHttp11ConnectionAsync(HttpRequestMessage request, CancellationToken cancellationToken)
   at System.Net.Http.HttpConnectionPool.GetHttpConnectionAsync(HttpRequestMessage request, CancellationToken cancellationToken)
   at System.Net.Http.HttpConnectionPool.SendWithRetryAsync(HttpRequestMessage request, Boolean doRequestAuth, CancellationToken   cancellationToken)
   at System.Net.Http.RedirectHandler.SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
   at System.Net.Http.DiagnosticsHandler.SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
   at MyAppNamespace.LoggingHandler.SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
```

When not sending any client certificates I did get the correct response back from the server indicating that there are no firewall/connection issues between deploy server and target server. It seems clear something goes wrong with the client certificates themselves.

When you google for these exceptions there's not a lot of useful information. One person with the exact same issue but without an answer. Several leads concerning managing supported SSL protocols. Several leads to certificate validation itself. No obvious solutions however and definitely nowhere an answer to why the call always fails after 1 minute 40.

This led to a series of test cases on a minimal repro to remove as much assumptions as possible and get to the actual issue at hand.

### Test case: ensure the deploy server can access the target server

Simplest test first, ensure the server itself can connect to the server, both without and with client certificates. This test was performed using curl while ssh'ed on the target machine. Both curl tests worked fine and the response was correct in both cases. Conclusion: something is off with dotnet itself as curl is able to properly handshake and authenticate.

```bash
  # note - using localhost here as replacement of actual endpoint

  # 1 - ensure without client certificate we get the correct error back
  curl -v https://localhost:58788/api/v1/init

  # 2 - ensure with certificate we get the correct error back
  curl -v --cert ./client.pem https://localhost:58788/api/v1/init
```

### Test case: disable all SSL validation possible.

```csharp
  // let our HttpClientHandler know all certificates are valid
  clientHandler.ServerCertificateCustomValidationCallback = (message, certificate2, arg3, arg4) => true;
  // don't load in revocation lists
  clientHandler.CheckCertificateRevocationList = false;
  // as far as I'm aware this doesn't do anything on .NET Core but better safe than sorry
  ServicePointManager.ServerCertificateValidationCallback = (message, certificate2, arg3, arg4) => true;
```

This did not make a difference - same exceptions still happen. In many ways it makes sense that this doesn't make a difference but was the obvious one to test first. These methods are purely focused on the server certificate, ie the SSL certificate of the server itself, before client certificate authentication is attempted. Unfortunately, while there is a whole range of server validation options, this is not the case for client certificates.

### Test case: certificate chain / root certificates

I tried two things here for this test case: loading in a pfx with the full certificate chain, ie including the intermediate and root certificate. And installing the intermediate/root certificates on the server.

I mainly lost a lot of time converting certificates to the correct formats on this one, the end result stayed the same unfortunately - same exceptions still happen with both cases. To be honest, I'm still not entirely sure if I did something wrong in this step. From what I read, this should have been enough. But in our case did not result in a working solution.

### Test case: configure supported SSL protocols

On the HttpClientHandler you can specify which SSL protocols it may use to negotiate the handshake with the server. Usually this should be left on default to ensure future protocols will also work but as some leads indicated this could be a possible issue, I ended up testing this out as well. The idea here was to basically test the same thing with a whole range of combinations to see if anything changed behaviour wise: stack trace / timings / ... As with above tests this did not make a difference (aside from where you only support obsolete protocols and it's normal that it doesn't work anymore).

```csharp
  // example test case
  clientHandler.SslProtocols = SslProtocols.Tls13 | SslProtocols.Tls12 | SslProtocols.Tls11 | SslProtocols.Tls;
```

### Consolidating results / google-fu

Putting all the facts together I started to focus a bit more on the actual timing of the issue - ie the fact that it always failed out after 100 seconds. Looking more into dotnet certificate issues on Linux and timeout/cancellation issues lead me to a [GitHub comment](https://github.com/dotnet/runtime/issues/31664#issuecomment-581654772) actually indicating that this 100 seconds was a default Linux timeout when certificate chains can't be retrieved. Based on this and all above information my conclusion was the following:

- our http call starts
- we pass in our client certificate
- dotnet validates server certificate
- dotnet tries to validate client certificate (not entirely sure what happens here and why curl doesn't fail here)
- dotnet goes to the certificate authority endpoint 
- the certificate authority endpoint is not accessible from our deploy server (unfortunately only found this out quite late in the process)
- call to ca times out after default Linux 100 seconds
- an exception is thrown by dotnet


With hindsight 20/20 and researching above exceptions again, I did find some other people running in the same issue with yet another stacktrace. For completeness sake I'll add that one here as well. [Original GitHub thread](https://github.com/dotnet/runtime/issues/805#issuecomment-567461955)

```
Unhandled exception. System.Threading.Tasks.TaskCanceledException: The operation was canceled.
   at System.Net.Http.ConnectHelper.EstablishSslConnectionAsyncCore(Stream stream, SslClientAuthenticationOptions sslOptions, CancellationToken cancellationToken)
   at System.Net.Http.HttpConnectionPool.ConnectAsync(HttpRequestMessage request, Boolean allowHttp2, CancellationTokencancellationToken)
   at System.Net.Http.HttpConnectionPool.CreateHttp11ConnectionAsync(HttpRequestMessage request, CancellationToken cancellationToken)
   at System.Net.Http.HttpConnectionPool.GetHttpConnectionAsync(HttpRequestMessage request, CancellationToken cancellationToken)
   at System.Net.Http.HttpConnectionPool.SendWithRetryAsync(HttpRequestMessage request, Boolean doRequestAuth, CancellationToken cancellationToken)
   at System.Net.Http.RedirectHandler.SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
   at System.Net.Http.HttpClient.FinishSendAsyncBuffered(Task`1 sendTask, HttpRequestMessage request, CancellationTokenSource cts, Boolean disposeCts)
   at TestHttpClient.Program.Main(String[] args)
   at TestHttpClient.Program.<Main>(String[] args)
```

# Our solution

After losing quite some time testing out all options, researching what dotnet actually does and browsing the collective internet knowledge, the eventual solution is extremely anticlimactic. A firewall change, ensuring that the certificate authority is actually reachable from your server, did the trick here. Hopefully other people who run into this issue can find this thread and short-circuit their frustrations towards a faster solution. I do hope that in the future we could get a better exception here and maybe also some manual validation mechanics as is already the case with normal SSL validation.


<br />
<br />

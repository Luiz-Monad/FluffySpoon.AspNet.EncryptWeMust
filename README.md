The simplest LetsEncrypt setup for ASP .NET Core. Almost no server configuration needed. 

`Install-Package FluffySpoon.AspNet.LetsEncrypt`
 
# Requirements
- Kestrel (which is default)
- ASP .NET Core 2.1+
- An always-on app-pool

## Getting an always-on app pool
This is required because the renewal job runs on a background thread and polls once every hour to see if the certificate needs renewal (this is a very cheap operation). 

It can be enabled using __just one__ the following techniques:
- Enabling Always On if using Azure App Service.
- Setting `StartMode` of the app pool to `AlwaysRunning` if using IIS.
- Hosting your ASP .NET Core application as a Windows Service.

# Usage example
If you want to try it yourself, you can also browse the sample project code here:

https://github.com/ffMathy/FluffySpoon.AspNet.LetsEncrypt/tree/master/src/FluffySpoon.AspNet.LetsEncrypt.Sample

## Configure the services
Add the following code to your `Startup` class' `ConfigureServices` method with real values instead of the sample values:

_Note that you can set either `TimeUntilExpiryBeforeRenewal`, `TimeAfterIssueDateBeforeRenewal` or both, but at least one of them has to be specified._

```csharp
//the following line adds the automatic renewal service.
services.AddFluffySpoonLetsEncryptRenewalService(new LetsEncryptOptions()
{
	Email = "some-email@github.com", //LetsEncrypt will send you an e-mail here when the certificate is about to expire
	UseStaging = false, //switch to true for testing
	Domains = new[] { DomainToUse },
	TimeUntilExpiryBeforeRenewal = TimeSpan.FromDays(30), //renew automatically 30 days before expiry
	TimeAfterIssueDateBeforeRenewal = TimeSpan.FromDays(7), //renew automatically 7 days after the last certificate was issued
	CertificateSigningRequest = new CsrInfo() //these are your certificate details
	{
		CountryName = "Denmark",
		Locality = "DK",
		Organization = "Fluffy Spoon",
		OrganizationUnit = "Hat department",
		State = "DK"
	}
});

//the following line tells the library to persist the certificate to a file, so that if the server restarts, the certificate can be re-used without generating a new one.
services.AddFluffySpoonLetsEncryptFileCertificatePersistence();

//the following line tells the library to persist challenges in-memory. challenges are the "/.well-known" URL codes that LetsEncrypt will call.
services.AddFluffySpoonLetsEncryptMemoryChallengePersistence();
```

## Inject the middleware
Inject the middleware in the `Startup` class' `Configure` method as such:

```csharp
public void Configure()
{
	app.UseFluffySpoonLetsEncryptChallengeApprovalMiddleware();
}
```

## Configure Kestrel to look for the certificate
Finally, to make Kestrel automatically select the LetsEncrypt certificate, we must configure it as such. Here's an example `Program.cs`:

```csharp
WebHost.CreateDefaultBuilder(args)
	.UseKestrel(kestrelOptions => kestrelOptions.ConfigureHttpsDefaults(
			httpsOptions => httpsOptions.ServerCertificateSelector = 
				(c, s) => LetsEncryptRenewalService.Certificate))
	.UseUrls("http://" + DomainToUse, "https://" + DomainToUse)
	.UseStartup<Startup>();
```

Tada! Your application now supports SSL via LetsEncrypt, even from the first HTTPS request. It will even renew your certificate automatically in the background.

# Optional: Configuring persistence
Persistence tells the middleware how to persist and retrieve the certificate, so that if the server restarts, the certificate can be re-used without generating a new one.

A certificate has a _key_ to distinguish between certificates, since there is both an account certificate and a site certificate that needs to be stored.

## File persistence
```csharp
services.AddFluffySpoonLetsEncryptFileCertificatePersistence();
services.AddFluffySpoonLetsEncryptFileChallengePersistence();
```

## Custom persistence
```csharp
services.AddFluffySpoonLetsEncryptCertificatePersistence(/* your own ILetsEncryptPersistence implementation */);
services.AddFluffySpoonLetsEncryptChallengePersistence(/* your own ILetsEncryptPersistence implementation */);

//you can also customize persistence via delegates.
services.AddFluffySpoonLetsEncryptCertificatePersistence(
	async (key, bytes) => File.WriteAllBytes("certificate_" + key, bytes),
	async (key) => File.ReadAllBytes("certificate_" + key, bytes));

//the same can be done for challenges with the same arguments.
services.AddFluffySpoonLetsEncryptChallengePersistence(...);
```

## Entity Framework persistence
Requires the NuGet package `FluffySpoon.AspNet.LetsEncrypt.EntityFramework`.

```csharp
// Certificate in this example is a database model class that has been configured with the database context.
class Certificate {
	[Key]
	public string Key { get; set; }

	public byte[] Bytes { get; set; }
}

//we only have to instruct how to add the certificate - `databaseContext.SaveChangesAsync()` is automatically called.
services.AddFluffySpoonLetsEncryptEntityFrameworkCertificatePersistence<DatabaseContext>(
	async (databaseContext, key, bytes) =>
	{
		var existingCertificate = databaseContext.Certificates.SingleOrDefault(x => x.Key == key);
		if (existingCertificate != null)
		{
			existingCertificate.Bytes = bytes;
		}
		else
		{
			databaseContext.Certificates.Add(new Certificate()
			{
				Key = key,
				Bytes = bytes
			});
		}
	},
	async (databaseContext, key) => databaseContext
		.Certificates
		.SingleOrDefault(x => x.Key == key)
		?.Bytes);

//the same can be done for challenges with the same arguments.
services.AddFluffySpoonLetsEncryptEntityFrameworkChallengePersistence<DatabaseContext>(...);
```

## Distributed cache (Redis etc) persistence
Requires:
- The NuGet package `FluffySpoon.AspNet.LetsEncrypt.DistributedCache`.
- A configured distributed cache in ASP .NET Core using the `services.AddDistributedRedisCache()` or similar.

```csharp
services.AddFluffySpoonLetsEncryptDistributedCertificatePersistence(expiry: TimeSpan.FromDays(30));
services.AddFluffySpoonLetsEncryptDistributedChallengePersistence(expiry: TimeSpan.FromHours(1));
```

# Hooking into events
You can register a an `ICertificateRenewalLifecycleHook` implementation which does something when certain events occur, as shown below. This can be useful if you need to notify a Slack channel or send an e-mail if an error occurs, or when the certificate has indeed been renewed.

```csharp
class MyLifecycleHook : ICertificateRenewalLifecycleHook {
	public async Task OnStartAsync() {
		//when the renewal background job has started.
	}

	public async Task OnStopAsync() {
		//when the renewal background job (or the application) has stopped.
		//this is not guaranteed to fire in critical application crash scenarios.
	}

	public async Task OnRenewalSucceededAsync() {
		//when the renewal has completed.
	}

	public async Task OnExceptionAsync(Exception error) {
		//when an error happened during the renewal process.
	}
}

//this is how to wire up the hook.
services.AddFluffySpoonLetsEncryptRenewalLifecycleHook<MyLifecycleHook>();
```

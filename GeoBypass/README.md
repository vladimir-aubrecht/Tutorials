# Goal
Avoid Geo restrictions from popular services like Hulu, Dinsey+ or HBO. Stream anything, anywhere from your network without installing anything on the device you are using for streaming.

Last part is crucial, because many people wants to stream on Xbox, Apple TV and other devices which doesn't allow to configure VPN.

The way around it is to configure VPN on the router. Be aware that this configuration is not redirecting all network traffic, but only traffic for bypassed streaming platforms.

In [here](GeoBypass.md) you can check how to configure it manually for Ubiquiti routers.

I prepared also [GeoBypass.json](GeoBypass.json) file with list of domains which needs to go through VPN to access the service. Be aware that at this moment json files doesn't need to be complete - I didn't put all shared domains from tutorial above there as I think they are not needed, but didn't verify it yet.

This json file will be eventually consumed by [RouterWizzard](https://github.com/vladimir-aubrecht/RouterWizzard). iOS app which will eventually allow you to configure your Ubiquiti router on few taps - it's work in progress, help is welcomed and I am opened to support of another routers too.
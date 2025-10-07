# Using a Vercel domain with Internet.nl

Running Internet.nl requires a full Django stack (PostgreSQL, Celery, Redis, RabbitMQ, ...).
Those services are not available on Vercel, so you cannot host the application purely on the
`*.vercel.app` platform. You **can** use a Vercel domain as a friendly entry point in front of
an existing Internet.nl deployment by proxying requests to your backend instance.

This document outlines the extra configuration that is needed once you have deployed the
application by following the regular [Docker deployment guide](Docker-deployment.md).

## 1. Prepare your Internet.nl backend

1. Deploy Internet.nl using Docker (see [Docker-deployment.md](Docker-deployment.md)) on a
   machine that is reachable from the public internet via HTTPS.
2. Make sure the deployment has a publicly trusted TLS certificate.
3. Add your Vercel hostname to the `ALLOWED_HOSTS` environment variable so that Django accepts
   requests forwarded by Vercel. For example:

   ```env
   ALLOWED_HOSTS=.internet.nl,internet.nl,internetnl.vercel.app
   ```

   The default environment files used for Docker Compose already expose the `ALLOWED_HOSTS`
   variable (see [`docker/defaults.env`](../docker/defaults.env)). Adding the Vercel domain to
   that list is sufficient.

## 2. Configure Vercel as a reverse proxy

Create a small Vercel project (for example by using an empty repository) and add a
`vercel.json` file with a rewrite rule that forwards all requests to your backend instance:

```json
{
  "rewrites": [
    {
      "source": "/(.*)",
      "destination": "https://your-backend.example.com/$1"
    }
  ],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Forwarded-Proto", "value": "https" }
      ]
    }
  ]
}
```

Deploy the project to Vercel, attach the `internetnl.vercel.app` (or your own custom) domain,
and verify that requests are correctly proxied to your backend. Because the Django settings
already trust the `HTTP_X_FORWARDED_PROTO` header (see [`internetnl/settings.py`](../internetnl/settings.py)),
HTTPS URLs will continue to be generated correctly for links inside the application.

## 3. Optional: add custom headers or authentication

If your deployment sits behind extra authentication (for example HTTP basic auth), you can add
additional headers inside the `vercel.json` configuration. Refer to the
[Vercel rewrites documentation](https://vercel.com/docs/edge-network/rewrites) for the full set
of options.

Once the rewrite is in place, your Vercel domain becomes a convenient façade in front of the
Docker-hosted Internet.nl instance.

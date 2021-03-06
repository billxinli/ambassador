# Ambassador Configuration

Ambassador is configured in a declarative fashion, using YAML manifests to describe the state of the world. As with Kubernetes, Ambassador's manifests are identified with `apiVersion`, `kind`, and `name`. The current `apiVersion` is `ambassador/v0`; currently-supported `kind`s are:

- [`Module`](modules) manifests configure things with can apply to Ambassador as a whole. For example, the `ambassador` module can define listener ports, and the `tls` module can configure TLS termination for Ambassador.

- [`AuthService`](modules#authservice) manifests configures the external authentication service[s] that Ambassador will use.

- [`Mapping`](mappings) manifests associate REST _resources_ with Kubernetes _services_. Ambassador _must_ have one or more mappings defined to provide access to any services at all.

## Configuration sources

Ambassador assembles its configuration from YAML blocks that may be stored:

- as `annotations` on Kubernetes `service`s (this is the recommended technique);
- as data in a Kubernetes `ConfigMap`; or
- as files in Ambassador's local filesystem.

The data contained within each YAML block is the same no matter where the blocks are stored, and multiple YAML documents are likewise supported no matter where the blocks are stored.

## Namespaces

Ambassador supports multiple namespaces within Kubernetes. To make this work correctly, you need to set the `AMBASSADOR_NAMESPACE` environment variable in Ambassador's container. By far the easiest way to do this is using Kubernetes' downward API (this is included in the YAML files from `getambassador.io`):

```yaml
        env:
        - name: AMBASSADOR_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace          
```

Given that `AMBASSADOR_NAMESPACE` is set, Ambassador [mappings](#mapping) can operate within the same namespace, or across namespaces. **Note well** that mappings will have to explictly include the namespace with the service to cross namespaces; see the [mapping](#mappings) documentation for more information.

If you only want Ambassador to only work within a single namespace, set `AMBASSADOR_SINGLE_NAMESPACE` as an environment variable.

## Reconfiguration Timing Configuration

Ambassador is constantly watching for changes to the service annotations. When changes are observed, Ambassador generates a new Envoy configuration and restarts the Envoy handling the heavy lifting of routing. Three environment variables provide control over the timing of this reconfiguration:

- `AMBASSADOR_RESTART_TIME` (default 5) sets the minimum number of seconds between restarts. No matter how often services are changed, Ambassador will never restart Envoy more frequently than this.

- `AMBASSADOR_DRAIN_TIME` (default 2) sets the number of seconds that the Envoy will wait for open connections to drain on a restart. Connections still open at the end of this time will be summarily dropped.

- `AMBASSADOR_SHUTDOWN_TIME` (default 3) sets the number of seconds that Ambassador will wait for the old Envoy to clean up and exit on a restart. **If Envoy is not able to shut down in this time, the Ambassador pod will exit.** If this happens, it is generally indicative of issues with restarts being attempted too often.

These environment variables can be set much like `AMBASSADOR_NAMESPACE`, above.

## Best Practices for Configuration

Ambassador's configuration is assembled from multiple YAML blocks, to help enable self-service routing and make it easier for multiple developers to collaborate on a single larger application. This implies a few things:

- Ambassador's configuration should be under version control.

    While you can always read back Ambassador's configuration from `annotation`s or its diagnostic service, Ambassador will not do versioning for you. Tools like [Forge](https://forge.sh) can help you maintain proper version control for your services' routing configurations.

- Be aware that Ambassador tries to not start with a broken configuration, but it's not perfect.

    Gross errors will result in Ambassador refusing to start, in which case `kubectl logs` will be helpful. However, it's always possible to e.g. map a resource to the wrong service, or use the wrong `rewrite` rules. Ambassador can't detect that on its own, although its diagnostic pages can help you figure it out.

- Be careful of mapping collisions.

    If two different developers try to map `/user/` to something, this can lead to unexpected behavior. Ambassador's canary-deployment logic means that it's more likely that traffic will be split between them than that it will throw an error -- again, the diagnostic service can help you here.

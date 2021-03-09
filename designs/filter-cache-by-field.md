Filter cache ListWatch by field
===================
## Motivation

The controller-runtime subscribe to changes using a cache mechanism that get
updated using kubernetes informers, those informers implement a `ListWatch`
that specify the namespace and type specified at controller builder time.

So the only possible filtering of that cache is namespace and resource type,
this means that if some user is interested only on Reconcile some resource with
specific fields it will still cache all the instances on that type increasing
the memory and CPU load. This also get worse in case controller-runtime is
deployed as a daemonset instead of a deployment since all the nodes with
daemonset's pods will expect to receive notification of all the instances and
that can end with scalation problems depending on the type or resource it's
watching.

The alternative that can be done with the current controller-runtme is
to implement a custom cache that do the filtering since cache is pluggable
into the controller-runtime [1]

This proposal is related to the following issue [2]

## Proposal

Incrase `cache.Options` as follow:

```golang
type Options struct {
	Scheme                  *runtime.Scheme
	Mapper                  meta.RESTMapper
	Resync                  *time.Duration
	Namespace               string
	FieldSelectorByResource map[schema.GroupResource]string
}
```

Incrase `manager.Options` as follow:

```golang
type Options struct {
	CacheFieldSelectorByResource map[schema.GroupResource]string
```

Pass the new manager options field field cache options and this
is passed to informer's ListWatch and add the filtering option:

```golang

# At pkg/cluster/cluster.go

cache, err := options.NewCache(config, cache.Options{Scheme: options.Scheme, Mapper: mapper, Resync: options.SyncPeriod, Namespace: options.Namespace, FieldSelectorByResource: options.CacheFieldSelectorByResource})


# At pkg/cache/internal/informers_map.go

func (ip *specificInformersMap) findFieldSelectorByGVR(gvr schema.GroupVersionResource) string {
	gr := schema.GroupResource{Group: gvr.Group, Resource: gvr.Resource}
	return ip.fieldSelectorByResource[gr]
}

ListFunc: func(opts metav1.ListOptions) (runtime.Object, error) {
			opts.FieldSelector = ip.findFieldSelectorByGVR(mapping.Resource)
...
```

Here is a PR with the implementatin at the `pkg/cache` part [3]

## Example

Users will change the Manager options, similar to namespace to specify what
Fields they want to limit the cache to by resource.

```golang
 ctrlOptions := ctrl.Options{
     ...
        CacheFieldSelectorByResource: map[schema.GroupResource]string{
							{Group: "", Resource: "nodes"}: "metadata.name=node01",
							{Group: "", Resource: "nodenetworkstate"}: "metadata.name=node01",
						},
		}
     ...
    }
```


[1] https://github.com/nmstate/kubernetes-nmstate/pull/687
[2] https://github.com/kubernetes-sigs/controller-runtime/issues/244
[3] https://github.com/kubernetes-sigs/controller-runtime/pull/1404

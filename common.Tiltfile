load('ext://namespace', 'namespace_create', 'namespace_inject')

def replace_namespace(yaml, new_namespace, forbidden_namespaces=['default', 'kube-system']):
    # read yaml 
    if type(yaml) == 'string':
        objects = read_yaml_stream(yaml)
    elif type(yaml) == 'blob':
        objects = decode_yaml_stream(yaml)
    else:
        fail('only takes string or blob, got: %s' % type(yaml))

    def _r(obj, path):
        for p in path[:-1]:
            if p in obj:
                obj = obj[p]
            else:
                return

        # check if
        # a) the key does not exist (-> unspecified hints at patchable namespace)
        # b) the key does exist and is not in the forbidden namespaces (e.g. "default")
        if (path[-1] not in obj) or (obj[path[-1]] not in forbidden_namespaces):
            obj[path[-1]] = new_namespace


    # inject the namespace
    for obj in objects:
        kind = obj['kind']

        _r(obj, ['metadata', 'namespace'])

        #
        # SPECIAL CASES
        #

        if kind == 'Deployment': # templates in deployments
            _r(obj, ['spec', 'template', 'metadata', 'namespace'])
        elif kind == 'RoleBinding': # subjects in role bindings
            for subject in obj['subjects']:
                _r(subject, ['namespace'])

    return encode_yaml_stream(objects)

def fix_names(yaml):
    # read yaml 
    if type(yaml) == 'string':
        objects = read_yaml_stream(yaml)
    elif type(yaml) == 'blob':
        objects = decode_yaml_stream(yaml)
    else:
        fail('only takes string or blob, got: %s' % type(yaml))

    for o in objects:
        if 'metadata' in o and 'name' in o['metadata'] and ':' in o['metadata']['name']:
            print('Replacing ":" in name %s (namespace: %s, kind: %s)' % (
                o['metadata']['name'],
                o['metadata']['namespace'] if 'namespace' in o['metadata'] else 'default',
                o['kind']
            ))
            o['metadata']['name'] = o['metadata']['name'].replace(':', '-')

    return encode_yaml_stream(objects)

def convert_to_tilt_id(resource_object):
    namespace = (
        resource_object['metadata']['namespace'] 
        if 'namespace' in resource_object['metadata'] 
        else 'default'
    )

    return '%s:%s:%s' % (
        resource_object['metadata']['name'], 
        resource_object['kind'], 
        namespace
    )

def is_handled_by_tilt(resource_object):
    # Tilt will automatically create a k8s_resource for specific resources:
    tilt_workload_resources = ['Deployment', 'Job', 'DaemonSet', "Service"]

    # special case: services are assigned by Tilt if there is a matching workload resource
    # if resource_object['kind'] == 'Service':
    #     workload_objects = [
    #         convert_to_tilt_id(r)
    #         for r in all_resources
    #         if r['kind'] in tilt_workload_resources
    #     ]

    #     for k in tilt_workload_resources:
    #         if convert_to_tilt_id(dict(resource_object, **{'kind': k})) in workload_objects:
    #             return True
    
    return resource_object['kind'] in tilt_workload_resources


def kustomize_resource(workload, kustomization_path, namespace, create_namespace=False, **kwargs):
    # all resources should be loaded to the given namespace and grouped in Tilt
    # -> load all resources with kustomize
    # -> inject the namespace to all resources
    # -> collect names of all resources using decode_yaml_stream
    resource_stream = kustomize(kustomization_path)
    resource_stream = replace_namespace(resource_stream, namespace)
    resource_stream = fix_names(resource_stream)
    resources = decode_yaml_stream(resource_stream)

    all_objects = [
        convert_to_tilt_id(r)
        for r in resources
        # Specific resources are already handled by Tilt's k8s_resource. Tilt will complain if 
        # we provide them in k8s_resource calls so we filter them here.
        if not is_handled_by_tilt(r)
    ]

    # add the namespace to the same Tilt group
    namespace_object = '{}:Namespace:default'.format(namespace)
    if create_namespace and namespace_object not in all_objects:
        # create the namespace
        namespace_create(namespace)
        all_objects.append(namespace_object)

    # deploy resources
    k8s_yaml(resource_stream)

    # Tilt will automatically create a k8s_resource for all resources listed in tilt_workload_resources
    # --> identify the workloads already created by Tilt
    # --> add the specified dependencies, labels, etc. to it.

    # identify
    workloads = [
        r['metadata']['name']
        for r in resources 
        if is_handled_by_tilt(r) and r['kind'] != "Service"
    ]

    # Tilt might have already created workloads automatically. If there is already a workload 
    # with the name specified in the parameter "workload", we add all objects to this workload.
    # Otherwise, we create a new workload.
    create_extra_workload = (workload not in workloads) and (len(all_objects) > 0)

    if create_extra_workload:
        k8s_resource(new_name=workload, objects=all_objects, **kwargs)

    for w in workloads:
        # The automatically created workloads might depend on other resources (e.g. namespaces, 
        # service accounts, etc.) from all_objects. Since we put those in a different workload,
        # we have to add a resource dependency here.
        modified_kwargs = {}
        if create_extra_workload:
            modified_kwargs['resource_deps'] = [workload]
        elif w == workload or (workload not in workloads and w == workloads[0]):
            modified_kwargs = kwargs
            modified_kwargs['objects'] = modified_kwargs.get('objects', []) + all_objects

        # all workloads should be put under the same label
        modified_kwargs['labels'] = kwargs.get('labels')

        k8s_resource(workload=w, **modified_kwargs)

    return resources
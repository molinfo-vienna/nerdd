load('ext://namespace', 'namespace_create', 'namespace_inject')
load('ext://git_resource', 'git_checkout')
load('ext://restart_process', 'custom_build_with_restart')

def replace_namespace(yaml, new_namespace, replaceable_namespaces=['default']):
    # read yaml 
    if type(yaml) == 'string':
        objects = read_yaml_stream(yaml)
    elif type(yaml) == 'blob':
        objects = decode_yaml_stream(yaml)
    else:
        fail('only takes string or blob, got: %s' % type(yaml))

    def _replace(obj, path):
        for p in path[:-1]:
            if p in obj:
                obj = obj[p]
            else:
                return

        # check if
        # a) the key does not exist (-> unspecified hints at patchable namespace)
        # b) the key does exist and its namespace is replaceable (e.g. "default")
        if (path[-1] not in obj) or (obj[path[-1]] in replaceable_namespaces):
            obj[path[-1]] = new_namespace


    # inject the namespace
    for obj in objects:
        kind = obj['kind']

        _replace(obj, ['metadata', 'namespace'])

        #
        # SPECIAL CASES
        #

        if kind == 'Deployment': # templates in deployments
            _replace(obj, ['spec', 'template', 'metadata', 'namespace'])
        elif kind == 'RoleBinding': # subjects in role bindings
            for subject in obj['subjects']:
                _replace(subject, ['namespace'])

    return encode_yaml_stream(objects)

def convert_to_tilt_id(resource_object):
    namespace = (
        resource_object['metadata']['namespace'] 
        if 'namespace' in resource_object['metadata'] 
        else 'default'
    )

    name = resource_object['metadata']['name'].replace(":", "\\:")
    kind = resource_object['kind'].replace(":", "\\:")
    namespace = namespace.replace(":", "\\:")

    return '%s:%s:%s' % (
        name, 
        kind, 
        namespace
    )

def is_handled_by_tilt(resource_object):
    # Tilt will automatically create a k8s_resource for specific resources:
    tilt_workload_resources = ['Deployment', 'Job', 'DaemonSet', "Service", "StatefulSet", "Pod"]

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


def kustomize_resource(
        workload, 
        kustomization_path, 
        namespace, 
        create_namespace=False, 
        replaceable_namespaces=['default'], 
        **kwargs
    ):
    # all resources should be loaded to the given namespace and grouped in Tilt
    # -> load all resources with kustomize
    # -> inject the namespace to all resources
    # -> collect names of all resources using decode_yaml_stream
    resource_stream = kustomize(kustomization_path)
    resource_stream = replace_namespace(resource_stream, namespace, replaceable_namespaces)
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

def _extract_dockerfile_entrypoint(dockerfile_path):
    entrypoint = []
    current = ""

    # add an empty line at the end to ensure that the last instruction is processed
    for raw_line in str(read_file(dockerfile_path)).splitlines() + ['']:
        line = raw_line.strip()
        if not line and not current:
            continue

        if line.startswith('#'):
            continue

        if line:
            if current:
                current += " " + line
            else:
                current = line

        if current.endswith('\\'):
            current = current[:-1].rstrip()
            continue

        instruction = current
        current = ""

        if not instruction.upper().startswith('ENTRYPOINT '):
            continue

        value = instruction[len('ENTRYPOINT '):].strip()
        if not value:
            continue

        if value.startswith('['):
            # value is something like ["executable", "param1", "param2"]
            docs = decode_yaml_stream(value)
            if len(docs) == 0 or type(docs[0]) != 'list':
                fail("Could not parse as JSON array: {}".format(value))

            entrypoint = [str(p) for p in docs[0]]
        else:
            # value is something like "executable param1 param2"
            entrypoint = [value]

    return entrypoint

def extract_entrypoint(kustomization_path, workload_name, dockerfile_path):
    """
    Extracts the effective entrypoint from a rendered Deployment/Job manifest.

    Resolution order:
    1. Kubernetes manifest command + args
    2. Dockerfile ENTRYPOINT + manifest args
    """
    yaml_stream = kustomize(kustomization_path)
    resources = decode_yaml_stream(yaml_stream)

    for r in resources:
        if r.get('kind') in ['Deployment', 'Job'] and r.get('metadata', {}).get('name') == workload_name:
            # Get the first container
            container = r['spec']['template']['spec']['containers'][0]

            def _to_list(s):
                if s == None:
                    return []

                return [str(p) for p in s]

            command = _to_list(container.get('command'))
            args = _to_list(container.get('args'))

            if command:
                return " ".join(command + args)

            image_entrypoint = _extract_dockerfile_entrypoint(dockerfile_path)
            if image_entrypoint:
                return " ".join(image_entrypoint + args)

            fail(
                (
                    "No command found for workload '%s': manifest has no command, "
                    + "and Dockerfile '%s' has no ENTRYPOINT"
                )
                % (workload_name, dockerfile_path)
            )

    fail("Could not find Deployment '%s' in '%s'" % (workload_name, kustomization_path))


def build_nerdd_module(
        project_name,
        repository_url,
        repository_name=None,
        dockerfile_path=None,
        live_update=False,
        live_update_module_and_link=False,
    ):
    """
    Checks out and builds a NERDD module. A typical NERDD module repository contains a Dockerfile
    to build the container image. If the Dockerfile supports restarts (e.g. Tilt can run touch,
    chmod, and sh), Tilt can restart the process when code changes are detected.

    When Live Update is enabled, this function synchronizes the project into the running
    container. It can additionally synchronize the ``nerdd-module`` and ``nerdd-link`` libraries
    so changes to those libraries are also reflected in the container.

    Parameters
    ----------
    project_name : str
        Name of the image and corresponding Kubernetes workload.
    repository_url : str
        Git repository to check out when the project directory is missing.
    repository_name : str or None, optional
        Name of the checkout directory under ``../../repos``. Can be useful if a repository
        contains multiple NERDD modules. If None, ``project_name`` is used. Defaults to None.
    dockerfile_path : str or None, optional
        Dockerfile to build. Defaults to ``Dockerfile`` in the project directory. Relative paths
        are resolved against the project directory; absolute paths are used unchanged.
    live_update : bool, optional
        Whether to build with process restart and synchronize the project into ``/app`` when code
        changes are detected. When false, uses a standard Docker build without Live Update.
    live_update_module_and_link : bool, optional
        Whether to watch and synchronize ``nerdd-module`` and ``nerdd-link`` into
        ``/deps/nerdd-module`` and ``/deps/nerdd-link`` during Live Update. Requires
        ``live_update`` to be true.
    """
    if live_update_module_and_link and not live_update:
        fail("live_update_module_and_link requires live_update=True")

    if repository_name == None:
        repository_name = project_name

    project_dir = '../../repos/{}'.format(repository_name)

    # checkout source code
    if not os.path.exists(project_dir):
        git_checkout(
            repository_url,
            checkout_dir=project_dir
        )

    if dockerfile_path == None:  # in starlark comparison with None happens with "=="
        dockerfile_path = 'Dockerfile'

    is_absolute_path = (
        dockerfile_path.startswith('/') or
        (
            os.name == 'nt' and
            (
                dockerfile_path.startswith('\\') or
                (
                    len(dockerfile_path) >= 3 and
                    dockerfile_path[1] == ':' and
                    dockerfile_path[2] in ['/', '\\']
                )
            )
        )
    )
    if not is_absolute_path:
        dockerfile_path = os.path.join(project_dir, dockerfile_path)

    if live_update:
        # build the docker image
        command = "docker buildx build -f {} -t $EXPECTED_REF --build-context repos={} .".format(
            dockerfile_path,
            # note: path is relative to context of Dockerfile, e.g. project_dir='../../repos/cypstrate'
            #       -> repos directory is the parent directory of project_dir
            os.path.join(project_dir, '..'),
        )

        deps = [project_dir, "."]
        live_update_steps = [sync(project_dir, "/app")]
        if live_update_module_and_link:
            deps += ["../../repos/nerdd-module", "../../repos/nerdd-link"]
            live_update_steps += [
                sync("../../repos/nerdd-module", "/deps/nerdd-module"),
                sync("../../repos/nerdd-link", "/deps/nerdd-link"),
            ]

        custom_build_with_restart(
            project_name,
            command=command,
            deps=deps,
            dir=project_dir,
            live_update=live_update_steps,
            entrypoint=extract_entrypoint('./envs/local/', project_name, dockerfile_path)
        )
    else:
        docker_build(
            project_name,
            context=project_dir,
            dockerfile=dockerfile_path,
        )

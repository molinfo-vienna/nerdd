load('ext://namespace', 'namespace_create')
load('ext://git_resource', 'git_checkout')

# limit the number of parallel updates
update_settings(max_parallel_updates=2)

# load config provided by the user via command line
MESSAGE_BROKER = 'kafka'

config.define_string("message_broker")
cfg = config.parse()
cfg['message_broker'] = cfg.get('message_broker', MESSAGE_BROKER)

# essential repositories used by many apps
base_repositories = [
    'nerdd-module',
    'nerdd-link',
]

# use comments to select services in order to improve speed
apps = [
    # openly accessible:
    'cypstrate',
    # 'cyplebrity',
    # 'hitdexter',
    # 'np-scout',
    # 'skin-doctor-cp',
    # 'skin-doctor-ternary',
    # 'trialblazer',

    # github permissions are necessary for these modules:
    # 'fame',
    # 'fame3r',
    # 'glory',
    # 'gloryx',

    # essential:
    'storage',
    'external-secrets',
    'gateway-api',
    'cert-manager',
    # 'haproxy-controller',
    'nginx-gateway',
    'keda',
    'rethinkdb',
    'nerdd-init-system',
    'nerdd-backend',
    'nerdd-frontend',
    'nerdd-process-jobs',
    'nerdd-serialize-jobs',

    # optional for monitoring:
    'argocd',
    # 'monitoring',
    # 'redpanda',
]

message_broker = cfg['message_broker']
if message_broker == "kafka":
    apps += ['strimzi', 'kafka']
elif message_broker == "rabbitmq":
    apps += ['rabbitmq-system', 'rabbitmq']

# check out base repositories
for repo in base_repositories:
    git_checkout(
        'https://github.com/molinfo-vienna/{}'.format(repo), 
        checkout_dir='./repos/{}'.format(repo),
        unsafe_mode=True,
    )

# create a namespace for apps
namespace_create('local')
k8s_resource(objects=['local'], labels=['infra'], new_name='namespace')

# load Tiltfiles of all specified apps
for app in apps:
    path_app_tiltfile = './apps/{}/Tiltfile'.format(app)
    if os.path.exists(path_app_tiltfile):
        include(path_app_tiltfile)
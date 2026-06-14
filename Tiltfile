load('ext://namespace', 'namespace_create')
load('ext://git_resource', 'git_checkout')

# 
# Define configuration options that can be set via command line when running `tilt up`, e.g.
# `tilt up -- --scrub-secrets=true --message-broker=kafka`
#

# message broker
DEFAULT_MESSAGE_BROKER = 'kafka'
config.define_string("message_broker")

# scrub secrets (replace secrets with [redacted] in Tilt UI)
DEFAULT_SCRUB_SECRETS = True
config.define_bool("scrub_secrets")

# number of parallel workflows to run when executing `tilt up`
DEFAULT_PARALLEL_UPDATES = 2
config.define_string("parallel_updates")

# load config provided by the user via command line
cfg = config.parse()
cfg['message_broker'] = cfg.get('message_broker', DEFAULT_MESSAGE_BROKER)
cfg['scrub_secrets'] = cfg.get('scrub_secrets', DEFAULT_SCRUB_SECRETS)
cfg['parallel_updates'] = int(cfg.get('parallel_updates', DEFAULT_PARALLEL_UPDATES))

# apply secret scrubbing settings
secret_settings(disable_scrub=not cfg['scrub_secrets'])

# limit the number of parallel updates
update_settings(max_parallel_updates=cfg['parallel_updates'])

#
# Define apps to be loaded
#

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
    'authelia',
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
    'routeboard',
]

message_broker = cfg['message_broker']
if message_broker == "kafka":
    apps += ['strimzi', 'kafka', 'kafka-ui']
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
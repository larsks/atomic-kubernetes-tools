#part-handler

import subprocess
import tempfile
import time

KUBE_SERVICES = [
    'etcd', 'kube-apiserver', 'kube-controller-manager', 'kube-scheduler',
    'kube-proxy', 'kubelet'
]

def list_types():
    return([
        'text/x-kube-pod',
        'text/x-kube-service',
        'text/x-kube-replica',
    ])

def start_kubernetes():
    for service in KUBE_SERVICES:
        subprocess.check_call(['systemctl', 'enable', service])
        subprocess.check_call(['systemctl', 'start', service])

    # wait for kubernetes to become available.
    while True:
        try:
            subprocess.check_call(['kubecfg', 'list', 'pods'])
            break
        except subprocess.CalledProcessError:
            time.sleep(1)

def create_object(objtype, payload):
    with tempfile.NamedTemporaryFile() as fd:
        fd.write(payload)
        fd.flush()
        subprocess.check_call(['kubecfg', '-c', fd.name,
                               'create', objtype])

def handle_part(data, ctype, filename, payload):
    if ctype == '__begin__':
        start_kubernetes()
    elif ctype == 'text/x-kube-pod':
        create_object('pods', payload)
    elif ctype == 'text/x-kube-service':
        create_object('services', payload)
    elif ctype == 'text/x-kube-replica':
        create_object('replicationControllers', payload)
    elif ctype == '__end__':
        pass
    else:
        raise ValueError(ctype)


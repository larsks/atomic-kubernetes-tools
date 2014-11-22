`kube-part-handler` is a custom part handler for use with [Fedora
Atomic][] 21 and [cloud-init][].  It allows you to pass in Kubernetes
configurations (for pods, services, and replicationControllers) to
`cloud-init` as part of a multi-part MIME user-data blob.

[cloud-init]: http://cloudinit.readthedocs.org/
[fedora atomic]: https://fedoraproject.org/wiki/Changes/Atomic_Cloud_Image

When this part handler is loaded, it will `start` and `enable` the
Kubernetes services.  It will then handle parts of the following MIME
types:

- `text/x-kube-pod`

   Parts of this type will be submitted use `kubecfg create pods`

- `text/x-kube-service`

   Parts of this type will be submitted use `kubecfg create services`

- `text/x-kube-replica`

   Parts of this type will be submitted use `kubecfg create
   replicationControllers`

## Tools

This repository includes a `write-mime-multipart.py` script to
generate MIME-multipart blobs suitable for use as `cloud-init`
user-data.  For example:

    python write-mime-multipart.py -o user-data \
      kube-part-handler \
      sample-pod.yaml

The script will attempt to guess the appropriate MIME type from the
first line of the file or from the file extension.  You can specify an
explicit MIME type by appending it to the filename separated by a
colon:

    python write-mime-multipart.py -o user-data \
      kube-part-handler \
      sample-pod.yaml \
      README.md:text/plain

In addition to the existing formats recognized by `cloud-init`, such
as `#!` for a shell script, or `#cloud-config` for a Cloud
configuration part, the `write-mime-multipart.py` script also
recognizes:

- `#kube-pod`
- `#kube-service`
- `#kube-replica`

## Example

Put the following in a file named `dbserver.yaml`:

    #kube-pod
    id: dbserver
    desiredState:
      manifest:
        version: v1beta1
        id: dbserver
        containers:
        - image: mysql
          name: dbserver
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: secret

Bundle this up with the custom part handler to create a user-data
blob:

    $ python write-mime-multipart.py -o user-data \
      kube-part-handler \
      disable-selinux.sh \
      dbserver.yaml

(The `disable-selinux.sh` script is only necessary until bug
[1166950][] is addressed.)

[1166950]: https://bugzilla.redhat.com/1166950

Boot an Atomic instance using that `user-data` file:

    $ nova boot --image fedora-atomic --key-name mykey \
      --flavor m1.small --user-data user-data kubetest

You will need to replace `fedora-atomc` with a name or UUID
appropriate for your environment, and you will need to replace `mykey`
with the name of your SSH key.  Depending on your environment you may
also need to assign the instance a floating ip before you can log in.

Log in as the `fedora` user (where `n.n.n.n` is the ip address of your
instance):

    $ ssh fedora@n.n.n.n

See that your pod has been scheduled:

    # kubecfg list pods
    ID                  Image(s)            Host                Labels              Status
    ----------          ----------          ----------          ----------          ----------
    dbserver            mysql               /                                       Waiting

At this point you will need to wait for Docker to pull the images.
After a while (where "a while" can mean several minutes), that should
transition to:

    # kubecfg list pods
    ID                  Image(s)            Host                Labels              Status
    ----------          ----------          ----------          ----------          ----------
    dbserver            mysql               127.0.0.1/                              Running

And `docker ps` should show something like:

    # docker ps
    CONTAINER ID        IMAGE                     COMMAND                CREATED             STATUS              PORTS               NAMES
    3b24ebc66d19        mysql:latest              "/entrypoint.sh mysq   4 minutes ago       Up 4 minutes                            k8s--dbserver.fd48803d--dbserver.etcd--3ae87bfb_-_7200_-_11e4_-_a7e8_-_fa163e2542b6--d8d6e972   
    c9df2911a1c5        kubernetes/pause:latest   "/pause"               7 minutes ago       Up 7 minutes                            k8s--net.d96a64a9--dbserver.etcd--3ae87bfb_-_7200_-_11e4_-_a7e8_-_fa163e2542b6--de5956cf        


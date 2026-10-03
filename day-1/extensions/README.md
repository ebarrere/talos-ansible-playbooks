# Extension configs

`ExtensionServiceConfig` documents applied by `tasks/apply-host-conf.yml`
(`talosctl patch mc`). Anything with a secret is committed **encrypted** as
`*.yaml.vault`; the playbook decrypts it to `*.yaml` at apply time, and the
plaintext is gitignored.

Sealed-secrets/kubeseal can't be used here: this is Talos machine config, not a
Kubernetes Secret — it's consumed by `machined` before the cluster exists.

    # edit the key, then:
    ansible-vault encrypt --output tailscale.yaml.vault tailscale.yaml
    git add tailscale.yaml.vault

Tailscale runs as a native Talos extension service (`talosctl services`,
`ext-tailscale`), not a pod.

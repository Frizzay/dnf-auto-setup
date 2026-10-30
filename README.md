# DNF Automatic Setup

Ansible role that sets up automatic updates on Fedora with `dnf5 automatic` and sends a notification to a Discord channel after every run. An interactive playbook is included, so you can use it without writing any YAML yourself.

## What the role does

- installs `dnf5-plugin-automatic`, `jq` and `curl`
- writes `/etc/dnf/automatic.conf` with the update type, reboot behaviour and Discord as the notification channel
- stores the webhook URL in `/etc/dnf/discord-webhook` (readable by root only)
- installs the script `/usr/local/bin/dnf-discord-notify`, which sends the report to Discord
- enables the systemd timer `dnf5-automatic.timer` and uses a drop-in to set the time it runs each day

The Discord message shows the hostname and time as a heading, followed by the report from dnf. Very long reports are truncated, because Discord allows at most 2000 characters per message.

## Requirements

- target systems running Fedora 41 or newer (dnf5)
- ansible-core 2.15 or newer on the control node
- SSH access with sudo privileges on the target systems
- a Discord webhook: Server Settings → Integrations → Webhooks → New Webhook, then copy the URL

## Repository layout

```
playbook.yml            interactive playbook that asks for all settings
roles/dnf_automatic/    the role itself
```

## Quick start: interactive playbook

Add your hosts to an inventory:

```ini
[fedora]
server1.example.com
server2.example.com
```

Run the playbook (add `-K` if sudo asks for a password):

```bash
ansible-playbook -i inventory playbook.yml
```

The playbook asks for the following values at startup:

| Question | Possible values | Default |
|---|---|---|
| Discord webhook URL | URL of the webhook (hidden input) | – |
| Which updates should be installed? | `security` (only updates with a security advisory), `default` (all) | `security` |
| Reboot automatically? | `never`, `when-needed` (e.g. after kernel updates), `when-changed` (after every update) | `never` |
| Also notify when there were no updates? | `y`, `n` | `n` |
| Time of the daily run | time in `HH:MM` format | `03:00` |
| Random delay in minutes | `0` (none) or a number, e.g. `30`, so that not all hosts hit the mirrors at the same time | `0` |
| Catch up on missed runs after boot? | `y`, `n` | `n` |
| Send a test message at the end? | `y`, `n` | `n` |

Values passed with `-e` are not prompted for, e.g. `-e dnf_automatic_reboot=when-needed`. Don't put the webhook URL on the command line, though, or it will end up in your shell history.

## Using the role in your own playbook

```yaml
- name: Set up automatic updates
  hosts: fedora
  become: true
  roles:
    - role: dnf_automatic
      vars:
        dnf_automatic_webhook_url: "{{ vault_discord_webhook_url }}"
        dnf_automatic_upgrade_type: default
        dnf_automatic_reboot: when-needed
        dnf_automatic_time: "04:30"
```

Keep the webhook URL in a Vault-encrypted file, for example `group_vars/fedora/vault.yml`, created with `ansible-vault create`.

## Role variables

| Variable | Default | Description |
|---|---|---|
| `dnf_automatic_webhook_url` | – (required) | Discord webhook URL the reports are sent to |
| `dnf_automatic_upgrade_type` | `security` | `security` or `default` (all updates) |
| `dnf_automatic_reboot` | `never` | `never`, `when-needed` or `when-changed` |
| `dnf_automatic_notify_no_updates` | `false` | also send a message when there were no updates |
| `dnf_automatic_time` | `"03:00"` | time of the daily run (`HH:MM`) |
| `dnf_automatic_random_delay` | `0` | random delay in minutes before each run |
| `dnf_automatic_catch_up` | `false` | catch up on runs missed while the machine was off |
| `dnf_automatic_send_test` | `false` | send a test message at the end of the role |
| `dnf_automatic_script` | `/usr/local/bin/dnf-discord-notify` | location of the notification script |
| `dnf_automatic_webhook_file` | `/etc/dnf/discord-webhook` | file the webhook URL is stored in |

The role validates all variables before it changes anything. Invalid values, such as an unknown update type or a time like `25:00`, stop the run with an error message. `ansible-doc -t role -r roles dnf_automatic` shows the documentation for all variables.

## Notes

- The role overwrites `/etc/dnf/automatic.conf` completely. Any custom changes there will be lost.
- On Fedora, many security fixes arrive as regular updates without a security advisory. These are not installed with `security`, so `default` is often the safer choice.
- Kernel updates only take effect after a reboot. With `never`, you need to reboot manually on a regular basis.
- If you enable catching up on missed runs, keep in mind: if the machine was off at the scheduled time, updates and any automatic reboot happen shortly after the next boot, which may be in the middle of the day.
- Automatic updates with reboots are usually a bad idea for prod servers. I might change my recommendation if AI keeps butt-blasting us with weekly 9.x CVEs...
- `/var/log/dnf5.log` shows what dnf automatic has installed. Check the timer and its next run with `systemctl list-timers '*automatic*'`.
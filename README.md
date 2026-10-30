# DNF Automatic Setup

Ansible playbook that sets up automatic updates on Fedora with `dnf5 automatic` and sends a notification to a Discord channel after every run.

## What the playbook does

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

## Usage

Add your hosts to an inventory:

```ini
[fedora]
server1.example.com
server2.example.com
```

Run the playbook (add `-K` if sudo asks for a password):

```bash
ansible-playbook -i inventory dnf-automatic.yaml
```

The playbook asks for the following values at startup:

| Question | Possible values | Default |
|---|---|---|
| Discord webhook URL | URL of the webhook (hidden input) | – |
| Which updates should be installed? | `security` (only updates with a security advisory), `default` (all) | `security` |
| Reboot automatically? | `never`, `when-needed` (e.g. after kernel updates), `when-changed` (after every update) | `never` |
| Also notify when there were no updates? | `y`, `n` | `n` |
| Time of the daily run | Time in `HH:MM` format | `03:00` |
| Random delay in minutes | `0` (none) or a number, e.g. `30`, so that not all hosts hit the mirrors at the same time | `0` |
| Catch up on missed runs after boot? | `y`, `n` | `n` |

Invalid input stops the playbook before anything on the system is changed.

### Test message

```bash
ansible-playbook -i inventory dnf-automatic.yaml --tags all,test
```

sets everything up and finishes by sending a test message to the channel.

### Without prompts

Values passed with `-e` are not prompted for. You might want to store the webhook url in a Vault-encrypted file if you do that though.

```bash
ansible-vault create secrets.yml
# Content: discord_webhook_url: https://discord.com/api/webhooks/...

ansible-playbook -i inventory dnf-automatic.yaml \
  -e dnf_auto_upgrade_type=security \
  -e dnf_auto_reboot=when-needed \
  -e dnf_auto_emit_no_updates=n \
  -e dnf_auto_time=03:00 \
  -e dnf_auto_random_delay=0 \
  -e dnf_auto_catch_up=n \
  -e @secrets.yml --ask-vault-pass
```

## Notes

- The playbook overwrites `/etc/dnf/automatic.conf` completely. Any custom changes there will be lost.
- On Fedora, many security fixes arrive as regular updates without a security advisory. These are not installed with `security`, so `default` is often the safer choice.
- Kernel updates only take effect after a reboot. With `never`, you need to reboot manually on a regular basis.
- If you enable catching up on missed runs, keep in mind: if the machine was off at the scheduled time, updates and any automatic reboot happen shortly after the next boot, which may be in the middle of the day.
- Automatic updates with reboots are usually a bad idea for prod servers. I might change my recommendation if AI keeps butt-blasting us with weekly 9.x CVEs...
- `/var/log/dnf5.log` shows what dnf automatic has installed. Check the timer and its next run with `systemctl list-timers '*automatic*'`.
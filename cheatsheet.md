\# Homelab Cheat Sheet



\## Git

git status                 # what changed?

git add .                  # stage all changes

git commit -m "message"    # save a snapshot

git push                   # send to GitHub



\## Proxmox

pct enter 101              # shell inside a container

pct status 101             # is it running?

qm status 100              # VM status



\## Troubleshooting

ip route / ping / cat /etc/resolv.conf   # bottom-up network check

ps -eo pid,etime,%cpu,cmd --forest       # what is a process waiting on?

Ctrl+Q                                    # unfreeze a paused terminal


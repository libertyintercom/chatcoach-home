# Install IMSUTH PAYPOINT (Paystack) on Ubuntu 24.04

Stack: Nginx (HTTPS) → uWSGI (Flask app) → MySQL 8, running under systemd.
Paystack sends payment events to `https://YOUR-DOMAIN/webhooks/paystack`, and the dashboard updates.

Replace `pay.hospital.example` everywhere below with your real domain.

---

## 0. Before you start

1. A fresh Ubuntu 24.04 server with a public IP and a sudo user (not root login).
2. A domain/subdomain with an **A record** pointing to the server IP (certbot needs this).
3. A Paystack account. You can test with test keys immediately. **Live keys need your business verified/activated by Paystack, with a settlement bank account added.**
4. This project's zip uploaded to the server (example: `scp imsuth_revenue_control_paystack.zip you@server:~`).

> What Paystack can report: money that passes through Paystack (checkout card/transfer/USSD, and transfers into Paystack dedicated virtual accounts). It cannot see cash, POS machines or deposits into an ordinary bank account.

---

## 1. System packages, timezone, firewall

```bash
sudo apt update && sudo apt -y upgrade
sudo apt install -y python3 python3-venv python3-dev build-essential libpcre3-dev libssl-dev \
                    mysql-server nginx certbot python3-certbot-nginx ufw unzip curl openssl

# "Today's revenue" uses the database clock, so set it to Nigerian time:
sudo timedatectl set-timezone Africa/Lagos

sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

Check: `python3 --version` shows 3.12.x and `sudo systemctl status mysql` shows *active (running)*.

---

## 2. Put the app in place

```bash
cd ~
unzip imsuth_revenue_control_paystack.zip

sudo adduser --system --group --no-create-home --home /opt/imsuth-revenue-control imsuth
sudo mkdir -p /opt/imsuth-revenue-control
sudo cp -r ~/imsuth_revenue_control/. /opt/imsuth-revenue-control/
sudo chown -R imsuth:imsuth /opt/imsuth-revenue-control
sudo chmod 755 /opt/imsuth-revenue-control
```

---

## 3. Database (least privilege)

Create the database and tables as the MySQL admin (on Ubuntu, `sudo mysql` works without a password):

```bash
cd /opt/imsuth-revenue-control
sudo mysql < schema.sql
```

Create the app's database user with only the permissions it needs. **Save the password it prints**, you need it in step 5.

```bash
DB_PASS=$(openssl rand -hex 24)
echo "MYSQL_PASSWORD=$DB_PASS"

sudo mysql <<SQL
CREATE USER 'imsuth_app'@'localhost' IDENTIFIED BY '$DB_PASS';
GRANT SELECT, INSERT, UPDATE ON imsuth_revenue.payments       TO 'imsuth_app'@'localhost';
GRANT SELECT, INSERT, UPDATE ON imsuth_revenue.webhook_events TO 'imsuth_app'@'localhost';
GRANT SELECT, INSERT         ON imsuth_revenue.audit_log      TO 'imsuth_app'@'localhost';
FLUSH PRIVILEGES;
SQL
```

The app can never delete payments or edit the audit log. MySQL listens on localhost only by default; leave it that way.

> Upgrading an existing Monnify-era database instead of a fresh install? Back it up, then run `sudo mysql < migrations/001_monnify_to_paystack.sql`. Skip this for a new install.

---

## 4. Python environment

```bash
cd /opt/imsuth-revenue-control
sudo -u imsuth python3 -m venv .venv
sudo -u imsuth .venv/bin/pip install --upgrade pip wheel
sudo -u imsuth .venv/bin/pip install -r requirements.txt
```

uWSGI compiles during this step and can take a minute or two. If it fails, re-check the `build-essential python3-dev libpcre3-dev libssl-dev` packages from step 1.

---

## 5. Configuration (secrets)

```bash
sudo mkdir -p /etc/imsuth
sudo cp /opt/imsuth-revenue-control/.env.example /etc/imsuth/imsuth.env
sudo chown root:imsuth /etc/imsuth/imsuth.env
sudo chmod 640 /etc/imsuth/imsuth.env

# Handy generators:
python3 -c "import secrets; print(secrets.token_urlsafe(48))"   # for APP_SECRET_KEY
openssl rand -base64 18 | tr -d '/+='                            # for APP_PASSWORD

sudo nano /etc/imsuth/imsuth.env
```

Set these values (no quotes, and avoid `$` and `#` in passwords):

| Key | Value |
|---|---|
| `APP_ENV` | `staging` for now (test keys). Becomes `production` at go-live (step 10) |
| `APP_SECRET_KEY` | the long random string you generated |
| `APP_USERNAME` / `APP_PASSWORD` | the dashboard login |
| `APP_BASE_URL` | `https://pay.hospital.example` (no trailing slash) |
| `COOKIE_SECURE` | `true` |
| `AUTO_INIT_DB` | `false` |
| `MYSQL_PASSWORD` | the password from step 3 |
| `PAYSTACK_SECRET_KEY` | `sk_test_...` from Paystack: **Settings → API Keys & Webhooks** |
| `PAYSTACK_FALLBACK_EMAIL` | a hospital mailbox. Paystack requires an email per payment; this is used when the cashier leaves it blank |

Never put the secret key in HTML/JavaScript or commit it anywhere. The same secret key signs webhooks; there is no separate webhook secret.

---

## 6. Run the app as a service

```bash
sudo cp /opt/imsuth-revenue-control/deployment/imsuth.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now imsuth

sudo systemctl status imsuth --no-pager
curl -s http://127.0.0.1:5000/health        # {"ok":true,...}
```

If it does not start: `sudo journalctl -u imsuth -n 50 --no-pager`.

---

## 7. Nginx and HTTPS

```bash
sudo cp /opt/imsuth-revenue-control/deployment/nginx.conf /etc/nginx/sites-available/imsuth
sudo sed -i 's/pay.your-domain.example/pay.hospital.example/g' /etc/nginx/sites-available/imsuth
sudo ln -s /etc/nginx/sites-available/imsuth /etc/nginx/sites-enabled/imsuth
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx

sudo certbot --nginx -d pay.hospital.example --redirect -m you@hospital.example --agree-tos
sudo systemctl list-timers | grep certbot       # auto-renewal is scheduled
```

Open `https://pay.hospital.example` and sign in. The pill at the top right should say **TEST MODE**.

The webhook path is restricted by Nginx to Paystack's three published IPs (52.31.139.75, 52.49.173.169, 52.214.14.220), and the app additionally checks the signature. Do not remove either layer. If you later put the site behind a proxy/CDN such as Cloudflare, the IP rule must be revisited.

---

## 8. Tell Paystack where to send events

Paystack Dashboard → **Settings → API Keys & Webhooks**:

- **Test Webhook URL:** `https://pay.hospital.example/webhooks/paystack`
- **Live Webhook URL:** `https://pay.hospital.example/webhooks/paystack` (set it now so go-live is one step less)
- Callback URL (optional; the app already sends its own): `https://pay.hospital.example/payment-return`

Save.

---

## 9. Test with test keys (do not skip)

1. In the dashboard, fill **Start a payment** (e.g. ₦5,000) and click *Create checkout*. Paystack checkout opens.
2. Pay with Paystack's test card: `4084 0840 8408 4081`, any future expiry, CVV `408`. If it asks for a PIN/OTP in test mode, use the values Paystack's checkout displays.
3. Within a few seconds the row should flip from **PENDING** to **PAID**.
4. Confirm in the database and logs:

```bash
sudo mysql imsuth_revenue -e "SELECT payment_reference,payment_status,amount_paid,payment_method,paystack_mode FROM payments ORDER BY id DESC LIMIT 5;"
sudo mysql imsuth_revenue -e "SELECT event_type,processing_status,error_message FROM webhook_events ORDER BY id DESC LIMIT 5;"
sudo journalctl -u imsuth -n 30 --no-pager
```

   `webhook_events.processing_status` should be `PROCESSED`.

5. Prove forged webhooks are rejected (run on the server, bypasses Nginx on purpose):

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST http://127.0.0.1:5000/webhooks/paystack -d '{"event":"charge.success","data":{"reference":"FAKE"}}'
# expected: 401
curl -s -o /dev/null -w "%{http_code}\n" -X POST http://127.0.0.1:5000/webhooks/paystack -H 'x-paystack-signature: deadbeef' -d '{}'
# expected: 401
```

6. Test the safety net for missed webhooks: start a payment, close the checkout without paying, and wait. After step 11 is installed, PENDING rows are re-checked every 5 minutes.

---

## 10. Go live

Only after step 9 works and Paystack has activated your business.

```bash
sudo systemctl stop imsuth

# Back up, then remove test data so live reports start clean:
sudo mysqldump --single-transaction imsuth_revenue | gzip > ~/before-live-$(date +%F).sql.gz
sudo mysql imsuth_revenue -e "DELETE FROM payments WHERE paystack_mode='test'; DELETE FROM webhook_events;"

sudo nano /etc/imsuth/imsuth.env
#   APP_ENV=production
#   PAYSTACK_SECRET_KEY=sk_live_...        (live key, from the Live tab)

sudo systemctl start imsuth
```

`APP_ENV=production` makes the app refuse to start with a test key, which protects against mixing them up. The pill should now say **LIVE PAYSTACK**. Make one small real payment, confirm it shows PAID, and check `webhook_events` shows `PROCESSED`. Confirm the money appears in Paystack's dashboard, and later in your settlement bank account (settlement is a separate Paystack step the dashboard does not prove).

---

## 11. Missed-webhook safety net (reconcile timer)

Webhooks can fail (server down, network). Paystack retries, but this also re-verifies anything still PENDING.

```bash
cd /opt/imsuth-revenue-control/deployment
sudo cp imsuth-reconcile.service imsuth-reconcile.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now imsuth-reconcile.timer

sudo systemctl start imsuth-reconcile.service
sudo journalctl -u imsuth-reconcile -n 5 --no-pager   # prints {"checked": N, "paid": ...}
```

---

## 12. Backups

```bash
sudo mkdir -p /var/backups/imsuth && sudo chmod 700 /var/backups/imsuth
sudo tee /root/.imsuth-backup.cnf >/dev/null <<'EOF'
[client]
user=root
socket=/var/run/mysqld/mysqld.sock
EOF
sudo chmod 600 /root/.imsuth-backup.cnf

sudo MYSQL_CNF=/root/.imsuth-backup.cnf MYSQL_DATABASE=imsuth_revenue \
     BACKUP_DEST=/var/backups/imsuth /opt/imsuth-revenue-control/deployment/backup_mysql.sh
```

Schedule it daily: `sudo crontab -e` and add

```
15 2 * * * MYSQL_CNF=/root/.imsuth-backup.cnf MYSQL_DATABASE=imsuth_revenue BACKUP_DEST=/var/backups/imsuth /opt/imsuth-revenue-control/deployment/backup_mysql.sh >> /var/log/imsuth-backup.log 2>&1
```

A backup on the same server is not enough. Copy `/var/backups/imsuth` to another machine or storage account every day (rsync/rclone), and practise a restore (`deployment/test_restore_mysql.sh`).

---

## Everyday operations

```bash
sudo journalctl -u imsuth -f                     # live app log
sudo tail -f /var/log/nginx/error.log
sudo systemctl restart imsuth                    # after editing /etc/imsuth/imsuth.env

# Update the code later:
sudo systemctl stop imsuth
sudo cp -r NEW_FILES/. /opt/imsuth-revenue-control/ && sudo chown -R imsuth:imsuth /opt/imsuth-revenue-control
sudo -u imsuth /opt/imsuth-revenue-control/.venv/bin/pip install -r /opt/imsuth-revenue-control/requirements.txt
sudo systemctl start imsuth
```

Rotating the Paystack key: generate the new key in Paystack, update `PAYSTACK_SECRET_KEY`, restart. Webhook signatures change with the key, so do it quickly.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Payment stays PENDING | Webhook URL not saved in Paystack, wrong mode (test URL vs live URL), or Nginx IP block. The reconcile timer fixes it within about 10 minutes. Check `journalctl -u imsuth` |
| Paystack reports webhook 401 | Secret key in `imsuth.env` does not match the mode sending the event (`sk_test_` vs `sk_live_`) |
| Paystack reports webhook 403 | Request did not come from a Paystack IP (proxy/CDN in front?) |
| `webhook_events.processing_status = ERROR` | Read `error_message`. Paystack will retry and the retry is re-processed; the row becomes PROCESSED when it succeeds |
| Status `REVIEW` | Paid amount differs from the amount the cashier entered. A person must check it in Paystack before treating it as revenue |
| Login works then immediately logs out | `COOKIE_SECURE=true` but you are using plain `http://`. Use the HTTPS URL |
| App won't start: "requires a live Paystack secret key" | `APP_ENV=production` with a `sk_test_` key. Use `staging` for tests |
| MySQL "Access denied" | Password in `imsuth.env` differs from step 3. `ALTER USER 'imsuth_app'@'localhost' IDENTIFIED BY 'new';` and update the file |
| uWSGI install fails | Install `build-essential python3-dev libpcre3-dev libssl-dev` and re-run step 4 |

---

## Before real staff use this with real patient/revenue data

`SECURITY_HARDENING.md` is still accurate: the app has **one shared admin login**. For a hospital finance system you need individual staff accounts, roles (cashier / finance / auditor / admin), login throttling and ideally MFA, plus a penetration test. Nginx rate-limits `/api/login` to 10 attempts per minute per IP as a stopgap, which is not a substitute.

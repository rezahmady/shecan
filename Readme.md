بالاخره علت دقیق مشخص شد: کارت شبکهٔ `eno1` تحت مدیریت `systemd-networkd` است، نه NetworkManager. پس لازم نیست مدیریت شبکه را عوض کنیم؛ برای فعال کردن شکن از همان `systemd-resolved` استفاده می‌کنیم.

این دستورها را یکجا اجرا کن. DNS شکن را روی رابط اصلی تنظیم می‌کنند و DNS عمومی را از تنظیمات فعال آن رابط حذف می‌کنند؛ تنظیمات قبلی هم برای `off` ذخیره می‌شوند.

```
cat > /usr/local/bin/shecan-dns <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

IFACE="eno1"
STATE="/var/lib/shecan-dns"
DNS1="178.22.122.101"
DNS2="185.51.200.1"

mkdir -p "$STATE"
chmod 700 "$STATE"

case "${1:-}" in
  on)
    if [ ! -f "$STATE/old-dns" ]; then
      resolvectl dns "$IFACE" > "$STATE/old-dns"
      resolvectl domain "$IFACE" > "$STATE/old-domain"
      resolvectl default-route "$IFACE" > "$STATE/old-route"
    fi

    resolvectl dns "$IFACE" "$DNS1" "$DNS2"
    resolvectl domain "$IFACE" "~."
    resolvectl default-route "$IFACE" yes

    echo "Shecan DNS enabled."
    resolvectl status "$IFACE"
    ;;
  off)
    if [ ! -f "$STATE/old-dns" ]; then
      echo "No saved DNS configuration. Nothing changed."
      exit 1
    fi

    # Restore the DNS values previously recorded for eno1.
    read -r _ old1 old2 _ < "$STATE/old-dns" || true
    if [ -n "${old1:-}" ]; then
      if [ -n "${old2:-}" ]; then
        resolvectl dns "$IFACE" "$old1" "$old2"
      else
        resolvectl dns "$IFACE" "$old1"
      fi
    else
      resolvectl revert "$IFACE"
    fi

    resolvectl revert "$IFACE"
    rm -f "$STATE/old-dns" "$STATE/old-domain" "$STATE/old-route"
    echo "Link settings reverted. Check the DNS inherited from Netplan."
    resolvectl status "$IFACE"
    ;;
  status)
    resolvectl status "$IFACE"
    ;;
  *)
    echo "Usage: shecan-dns {on|off|status}"
    exit 2
    ;;
esac
EOF

chmod 700 /usr/local/bin/shecan-dns
```

بعد برای فعال کردن:

```
shecan-dns on
```

و برای بررسی:

```
shecan-dns status
```

برای خاموش کردن:

```
shecan-dns off
```

توجه: این نسخه فقط DNS را در `systemd-resolved` تغییر می‌دهد؛ تنظیم پایدار Netplan و درخواست اعلام IP حرفه‌ای را تغییر نمی‌دهد. همچنین برای `off` بهتر است قبل از استفاده، ذخیره و بازیابی دقیق تنظیمات را اصلاح کنیم؛ چون DNS اولیه در Netplan برابر `8.8.8.8` بوده و ممکن است پس از بازگردانی موقت، دوباره از Netplan اعمال شود.

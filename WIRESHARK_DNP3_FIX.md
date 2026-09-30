# Fix: make the noVNC Wireshark see the farm↔turbine DNP3

**Problem (seen last year):** students couldn't see DNP3 traffic for the wind farm in the noVNC Wireshark, and had to fall back to `tshark`.

**Root cause:** the `wind-farm` repo's `docker-compose.yml` has **only an `hmi` service — there is no Wireshark/noVNC container at all**. Students were using the *turbine's* noVNC Wireshark, which runs `network_mode: host` and therefore can't see the farm↔turbine DNP3. That DNP3 rides **Tailscale inside the farm `hmi` container's own network namespace** (`tailscaled` runs in that container), so nothing on the host or in another container sees it.

**Fix:** add a Wireshark/noVNC service to the wind-farm compose that **shares the `hmi` container's network namespace** with `network_mode: "service:hmi"` — so Wireshark sees `tailscale0` and can live-capture DNP3 (filter `dnp3`). Note this differs from the turbine, which uses `network_mode: host` (the turbine's Modbus is on host interfaces; the farm's DNP3 is on the hmi container's tailnet).

> ⚠️ **Test before merging.** The `network_mode: "service:hmi"` approach is the correct namespace, but confirm on a run that the noVNC Wireshark lists `tailscale0` and captures DNP3. If the published `tools` image's baked-in supervisor config doesn't auto-start Wireshark in this mode, the two `.conf` volumes below pin it.

---

## 1. Add to `docker-compose.yml` (a new service under `services:`)

```yaml
  wireshark:
    image: ghcr.io/patsec/wind-turbine/tools:main
    init: true
    privileged: true            # required for capturing traffic
    network_mode: "service:hmi" # share the farm hmi container's namespace so Wireshark sees tailscale0 / DNP3
    depends_on:
      - hmi
    volumes:
      - ./configs/docker/tigervnc-wireshark.conf:/etc/supervisor/conf.d/tigervnc-wireshark.conf
      - ./configs/docker/wireshark.conf:/etc/supervisor/conf.d/wireshark.conf
```

(The farm's existing `hmi` service is unchanged. Because the wireshark service joins the hmi namespace, its noVNC web port is exposed via the `hmi` service — add `- 8080:8080` to the `hmi` service's `ports:` list, next to `1880:1880`, so the noVNC view is reachable/forwarded in Codespaces.)

## 2. Add `configs/docker/wireshark.conf`

```
[program:wireshark]
priority=1
environment=DISPLAY=:1
command=/usr/bin/wireshark
autorestart=false
stdout_logfile=/dev/fd/1
stdout_logfile_maxbytes=0
redirect_stderr=true
```

## 3. Add `configs/docker/tigervnc-wireshark.conf`

```
[program:x11]
priority=0
command=/usr/bin/Xtigervnc -desktop "Wireshark" -localhost -rfbport 5900 -SecurityTypes None -AlwaysShared -AcceptKeyEvents -AcceptPointerEvents -AcceptSetDesktopSize -SendCutText -AcceptCutText :1
autorestart=true
stdout_logfile=/dev/fd/1
stdout_logfile_maxbytes=0
redirect_stderr=true
```

## 4. (Optional) `.devcontainer/devcontainer.json`

Ensure the noVNC port (8080) is forwarded so students can open the Wireshark GUI in the browser, the same way the turbine repo forwards it.

---

## Result
With this in place, in the farm Codespace students open the noVNC Wireshark, pick the `tailscale0` interface, and filter `dnp3` to watch the farm poll each turbine live — no tshark detour needed. The `tshark` method stays in Lab 2 as a reliable fallback and for producing a `.pcap` to open in Wireshark.

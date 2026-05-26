import os
import sys
import json
import shutil
import subprocess
import ctypes
import random
import threading
import time
import urllib.request
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from pathlib import Path
try:
    import winreg
except Exception:
    winreg = None

URL = "https://sbxh2.com/"
APP_NAME = "NTKViewer"


def get_base_dir():
    local_appdata = os.environ.get("LOCALAPPDATA")
    if local_appdata:
        return Path(local_appdata) / APP_NAME
    return Path.home() / APP_NAME


BASE_DIR = get_base_dir()
EXT_DIR = BASE_DIR / "ntk_masker_ext"
PROFILE_DIR = BASE_DIR / "edge_auto_profile"
PASS_FLAG = BASE_DIR / "cloudflare_passed.flag"

EDGE_PATHS = [
    r"C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe",
    r"C:\Program Files\Microsoft\Edge\Application\msedge.exe",
]

IP_SERVER_PORT = 17862
IP_SERVER_URL = f"http://127.0.0.1:{IP_SERVER_PORT}/change-ip"
NET_CLASS_KEY = r"SYSTEM\CurrentControlSet\Control\Class\{4d36e972-e325-11ce-bfc1-08002be10318}"
_ip_server = None


def is_admin():
    try:
        return ctypes.windll.shell32.IsUserAnAdmin()
    except Exception:
        return False


def run_cmd(cmd, timeout=70):
    try:
        p = subprocess.run(
            cmd,
            shell=True,
            capture_output=True,
            text=True,
            encoding="cp949",
            errors="ignore",
            timeout=timeout
        )
        return p.returncode, ((p.stdout or "") + (p.stderr or "")).strip()
    except Exception as e:
        return -1, str(e)


def ps_quote(s):
    return str(s).replace("'", "''")


def get_public_ip():
    for url in ("https://api.ipify.org", "https://icanhazip.com", "https://ifconfig.me/ip"):
        try:
            with urllib.request.urlopen(url, timeout=8) as r:
                return r.read().decode("utf-8", errors="ignore").strip()
        except Exception:
            pass
    return ""


def get_default_adapter():
    cmd = r'''powershell -NoProfile -ExecutionPolicy Bypass -Command "$r = Get-NetRoute -DestinationPrefix '0.0.0.0/0' | Sort-Object RouteMetric | Select-Object -First 1; if ($r) { Get-NetAdapter -InterfaceIndex $r.InterfaceIndex | Select-Object Name, InterfaceDescription, MacAddress, Status, InterfaceGuid, InterfaceIndex | ConvertTo-Json -Compress }"'''
    code, out = run_cmd(cmd)
    if code != 0 or not out.strip():
        raise RuntimeError("adapter_not_found")

    data = json.loads(out.strip().splitlines()[-1].strip())
    return {
        "name": data.get("Name", ""),
        "desc": data.get("InterfaceDescription", ""),
        "mac": data.get("MacAddress", ""),
        "status": data.get("Status", ""),
        "guid": str(data.get("InterfaceGuid", "")).strip("{}").lower(),
        "index": str(data.get("InterfaceIndex", "")),
    }


def random_mac():
    first_choices = [0x00, 0x04, 0x08, 0x0C, 0x10, 0x14, 0x18, 0x1C, 0x20, 0x24, 0x28, 0x2C]
    b = [random.choice(first_choices)] + [random.randint(0, 255) for _ in range(5)]
    return "".join(f"{x:02X}" for x in b)


def pretty_mac(mac):
    mac = str(mac or "").replace("-", "").replace(":", "").upper()
    if len(mac) != 12:
        return mac
    return "-".join(mac[i:i + 2] for i in range(0, 12, 2))


def find_adapter_registry_key(interface_desc, interface_guid):
    if winreg is None:
        raise RuntimeError("winreg_unavailable")

    root = winreg.OpenKey(
        winreg.HKEY_LOCAL_MACHINE,
        NET_CLASS_KEY,
        0,
        winreg.KEY_READ | winreg.KEY_WRITE | getattr(winreg, "KEY_WOW64_64KEY", 0)
    )

    fallback = []
    i = 0

    while True:
        try:
            sub = winreg.EnumKey(root, i)
            i += 1
        except OSError:
            break

        if not sub.isdigit():
            continue

        path = NET_CLASS_KEY + "\\" + sub

        try:
            key = winreg.OpenKey(
                winreg.HKEY_LOCAL_MACHINE,
                path,
                0,
                winreg.KEY_READ | winreg.KEY_WRITE | getattr(winreg, "KEY_WOW64_64KEY", 0)
            )

            driver_desc = ""
            netcfg = ""

            try:
                driver_desc, _ = winreg.QueryValueEx(key, "DriverDesc")
            except FileNotFoundError:
                pass

            try:
                netcfg, _ = winreg.QueryValueEx(key, "NetCfgInstanceId")
            except FileNotFoundError:
                pass

            driver_desc = str(driver_desc)
            netcfg_clean = str(netcfg).strip("{}").lower()
            target_guid = str(interface_guid).strip("{}").lower()

            if target_guid and netcfg_clean == target_guid:
                winreg.CloseKey(key)
                winreg.CloseKey(root)
                return path

            d1 = driver_desc.lower()
            d2 = str(interface_desc).lower()

            if d1 == d2 or d1 in d2 or d2 in d1:
                fallback.append(path)

            winreg.CloseKey(key)
        except OSError:
            continue

    winreg.CloseKey(root)

    if fallback:
        return fallback[0]

    raise RuntimeError("registry_key_not_found")


def set_network_address(reg_path, mac12):
    key = winreg.OpenKey(
        winreg.HKEY_LOCAL_MACHINE,
        reg_path,
        0,
        winreg.KEY_SET_VALUE | getattr(winreg, "KEY_WOW64_64KEY", 0)
    )
    winreg.SetValueEx(key, "NetworkAddress", 0, winreg.REG_SZ, mac12)
    winreg.CloseKey(key)


def restart_adapter(adapter_name):
    name = ps_quote(adapter_name)
    run_cmd(f'''powershell -NoProfile -ExecutionPolicy Bypass -Command "Disable-NetAdapter -Name '{name}' -Confirm:$false"''', timeout=35)
    time.sleep(5)
    run_cmd(f'''powershell -NoProfile -ExecutionPolicy Bypass -Command "Enable-NetAdapter -Name '{name}' -Confirm:$false"''', timeout=35)
    time.sleep(10)


def renew_ip():
    run_cmd("ipconfig /flushdns", timeout=20)
    run_cmd("ipconfig /release", timeout=45)
    time.sleep(4)
    run_cmd("ipconfig /renew", timeout=75)
    time.sleep(10)


def perform_ip_change():
    before_ip = get_public_ip()
    adapter = get_default_adapter()
    reg_path = find_adapter_registry_key(adapter["desc"], adapter["guid"])
    new_mac = random_mac()

    set_network_address(reg_path, new_mac)
    restart_adapter(adapter["name"])
    renew_ip()

    after_adapter = get_default_adapter()
    after_ip = get_public_ip()

    return {
        "ok": True,
        "before_ip": before_ip,
        "after_ip": after_ip,
        "before_mac": adapter.get("mac", ""),
        "after_mac": after_adapter.get("mac", ""),
        "target_mac": pretty_mac(new_mac),
        "ip_changed": bool(before_ip and after_ip and before_ip != after_ip),
        "mac_changed": str(after_adapter.get("mac", "")).upper() == pretty_mac(new_mac).upper()
    }


def run_ip_helper(result_path):
    try:
        BASE_DIR.mkdir(parents=True, exist_ok=True)
        result = perform_ip_change()
    except Exception as e:
        result = {
            "ok": False,
            "error": str(e)
        }

    try:
        Path(result_path).write_text(json.dumps(result, ensure_ascii=False), encoding="utf-8")
    except Exception:
        pass


def run_ip_helper_elevated():
    BASE_DIR.mkdir(parents=True, exist_ok=True)
    result_path = BASE_DIR / "ip_change_result.json"

    try:
        result_path.unlink()
    except Exception:
        pass

    if is_admin():
        return perform_ip_change()

    if getattr(sys, "frozen", False):
        exe = sys.executable
        params = f'--ip-helper "{result_path}"'
    else:
        exe = sys.executable
        params = f'"{Path(__file__).resolve()}" --ip-helper "{result_path}"'

    rc = ctypes.windll.shell32.ShellExecuteW(None, "runas", exe, params, None, 0)

    if rc <= 32:
        return {
            "ok": False,
            "error": "uac_cancelled"
        }

    deadline = time.time() + 150

    while time.time() < deadline:
        if result_path.exists():
            try:
                return json.loads(result_path.read_text(encoding="utf-8"))
            except Exception as e:
                return {
                    "ok": False,
                    "error": str(e)
                }
        time.sleep(0.6)

    return {
        "ok": False,
        "error": "timeout"
    }



def reset_extension_flag():
    try:
        if EXT_DIR.exists():
            shutil.rmtree(EXT_DIR, ignore_errors=True)
    except Exception:
        pass

    try:
        if PASS_FLAG.exists():
            PASS_FLAG.unlink()
    except Exception:
        pass


def schedule_reset_and_exit():
    def worker():
        time.sleep(1.2)
        reset_extension_flag()
        try:
            info_box("재시작 필요", "정리 완료.\n다시 실행해주세요.")
        except Exception:
            pass
        try:
            kill_edge_once()
        except Exception:
            pass
        os._exit(0)

    threading.Thread(target=worker, daemon=True).start()

class IPChangeHandler(BaseHTTPRequestHandler):
    def send_cors(self):
        self.send_header("Access-Control-Allow-Origin", "*")
        self.send_header("Access-Control-Allow-Methods", "POST, OPTIONS")
        self.send_header("Access-Control-Allow-Headers", "Content-Type")

    def do_OPTIONS(self):
        self.send_response(204)
        self.send_cors()
        self.end_headers()

    def do_POST(self):
        if self.path != "/change-ip":
            self.send_response(404)
            self.send_cors()
            self.end_headers()
            return

        result = run_ip_helper_elevated()
        reset_extension_flag()
        result["restart"] = True
        body = json.dumps(result, ensure_ascii=False).encode("utf-8")

        self.send_response(200)
        self.send_cors()
        self.send_header("Content-Type", "application/json; charset=utf-8")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)
        schedule_reset_and_exit()

    def log_message(self, *args):
        return


def start_ip_server():
    global _ip_server

    if _ip_server is not None:
        return True

    try:
        _ip_server = ThreadingHTTPServer(("127.0.0.1", IP_SERVER_PORT), IPChangeHandler)
        t = threading.Thread(target=_ip_server.serve_forever, daemon=True)
        t.start()
        return True
    except Exception:
        _ip_server = None
        return False


def msedge_running():
    code, out = run_cmd('tasklist /FI "IMAGENAME eq msedge.exe" /NH', timeout=10)
    return "msedge.exe" in out.lower()


def wait_for_edge_exit():
    time.sleep(4)
    while msedge_running():
        time.sleep(2)


CONTENT_JS = r"""
(() => {
  const host = location.hostname || '';

  if (!host.includes('sbxh2') && !host.includes('whoas')) {
    return;
  }



  function installDevtoolsPreflightShield() {
    try {
      const DEV_WARN_KEY = 'ntk_dev_warn';
      const blockedUrl = url => String(url || '').includes('/api/dev-block');

      try { localStorage.removeItem(DEV_WARN_KEY); } catch (_) {}
      try { sessionStorage.removeItem(DEV_WARN_KEY); } catch (_) {}

      try {
        Object.defineProperty(window, '__ntkDevtoolsPreflight', {
          configurable: true,
          get() { return 1; },
          set(_) { return true; }
        });
      } catch (_) {
        try { window.__ntkDevtoolsPreflight = 1; } catch (__) {}
      }

      try {
        Object.defineProperty(window, '__ntkDevtoolsTripped', {
          configurable: true,
          get() { return undefined; },
          set(_) { return true; }
        });
      } catch (_) {}

      try {
        Object.defineProperty(window, 'devtoolsFormatters', {
          configurable: true,
          get() { return []; },
          set(_) { return true; }
        });
      } catch (_) {
        try { window.devtoolsFormatters = []; } catch (__) {}
      }

      try {
        const rawSetItem = Storage.prototype.setItem;
        Storage.prototype.setItem = function(key, value) {
          if (String(key) === DEV_WARN_KEY) return;
          return rawSetItem.call(this, key, value);
        };
      } catch (_) {}

      try {
        const rawFetch = window.fetch;
        if (rawFetch && !rawFetch.__ntkPatched) {
          const patchedFetch = function(input, init) {
            let url = '';
            try { url = typeof input === 'string' ? input : (input && input.url) || ''; } catch (_) {}
            if (blockedUrl(url)) {
              return Promise.resolve(new Response('{}', { status: 204, headers: { 'content-type': 'application/json' } }));
            }
            return rawFetch.apply(this, arguments);
          };
          patchedFetch.__ntkPatched = true;
          window.fetch = patchedFetch;
        }
      } catch (_) {}

      try {
        const rawBeacon = navigator.sendBeacon && navigator.sendBeacon.bind(navigator);
        if (rawBeacon && !navigator.sendBeacon.__ntkPatched) {
          const patchedBeacon = function(url, data) {
            if (blockedUrl(url)) return true;
            return rawBeacon(url, data);
          };
          patchedBeacon.__ntkPatched = true;
          navigator.sendBeacon = patchedBeacon;
        }
      } catch (_) {}

      try {
        const rawOpen = XMLHttpRequest.prototype.open;
        const rawSend = XMLHttpRequest.prototype.send;
        if (!XMLHttpRequest.prototype.__ntkDevBlockPatched) {
          XMLHttpRequest.prototype.__ntkDevBlockPatched = true;
          XMLHttpRequest.prototype.open = function(method, url) {
            this.__ntkBlockDevUrl = blockedUrl(url);
            if (this.__ntkBlockDevUrl) {
              arguments[1] = 'data:application/json,{}';
            }
            return rawOpen.apply(this, arguments);
          };
          XMLHttpRequest.prototype.send = function() {
            if (this.__ntkBlockDevUrl) {
              try { this.abort(); } catch (_) {}
              return;
            }
            return rawSend.apply(this, arguments);
          };
        }
      } catch (_) {}

      const looksLikeDevtoolsCode = handler => {
        try {
          const src = typeof handler === 'string' ? handler : Function.prototype.toString.call(handler);
          return src.includes('ntkDevtoolsPreflight') ||
                 src.includes('checkDevTools') ||
                 src.includes('devtoolsFormatters') ||
                 src.includes('auto:debugger') ||
                 src.includes('auto:formatters') ||
                 src.includes('ntk_dev_warn') ||
                 src.includes('/api/dev-block');
        } catch (_) {
          return false;
        }
      };

      try {
        const rawSetTimeout = window.setTimeout;
        const rawSetInterval = window.setInterval;

        if (!window.__ntkTimerShieldPatched) {
          window.__ntkTimerShieldPatched = true;

          window.setTimeout = function(handler, timeout, ...args) {
            if (looksLikeDevtoolsCode(handler)) return 0;
            return rawSetTimeout(handler, timeout, ...args);
          };

          window.setInterval = function(handler, timeout, ...args) {
            if (looksLikeDevtoolsCode(handler)) return 0;
            return rawSetInterval(handler, timeout, ...args);
          };
        }
      } catch (_) {}

      try {
        const rawEval = window.eval;
        window.eval = function(code) {
          if (typeof code === 'string') {
            code = code
              .replace(/\bdebugger\b\s*;?/g, '')
              .replace(/ntk_dev_warn/g, 'ntk_dev_warn_disabled');
          }
          return rawEval.call(this, code);
        };
      } catch (_) {}

      try {
        const RawFunction = window.Function;
        if (!RawFunction.__ntkDevShieldPatched) {
          const PatchedFunction = new Proxy(RawFunction, {
            apply(target, thisArg, args) {
              const cleaned = args.map(arg => String(arg).replace(/\bdebugger\b\s*;?/g, ''));
              return Reflect.apply(target, thisArg, cleaned);
            },
            construct(target, args) {
              const cleaned = args.map(arg => String(arg).replace(/\bdebugger\b\s*;?/g, ''));
              return Reflect.construct(target, cleaned);
            }
          });
          PatchedFunction.__ntkDevShieldPatched = true;
          window.Function = PatchedFunction;
        }
      } catch (_) {}

      function killDevOverlays() {
        try { localStorage.removeItem(DEV_WARN_KEY); } catch (_) {}
        try { sessionStorage.removeItem(DEV_WARN_KEY); } catch (_) {}
        document.querySelectorAll('#ntk_devtools_overlay, [id*="devtools_overlay"], [id*="ntk_devtools"], div[role="dialog"][aria-modal="true"]').forEach(el => {
          const t = el.textContent || '';
          if (el.id === 'ntk_devtools_overlay' || t.includes('개발자 도구 차단') || t.includes('접근이 차단되었습니다')) {
            try { el.remove(); } catch (_) {}
          }
        });
        try { document.documentElement.style.setProperty('overflow', 'auto', 'important'); } catch (_) {}
        try { document.body && document.body.style.setProperty('overflow', 'auto', 'important'); } catch (_) {}
      }

      try {
        const rawAppend = Node.prototype.appendChild;
        const rawInsert = Node.prototype.insertBefore;
        const rawPrepend = Element.prototype.prepend;

        const neuter = node => {
          try {
            if (node && node.tagName === 'SCRIPT') {
              const id = node.id || '';
              const txt = node.textContent || '';
              const src = node.src || '';
              if (id === 'ntk-devtools-preflight' || txt.includes('ntkDevtoolsPreflight') || txt.includes('/api/dev-block')) {
                node.type = 'javascript/blocked';
                node.textContent = '';
                try { node.removeAttribute('src'); } catch (_) {}
              }
            }
          } catch (_) {}
          return node;
        };

        if (!Node.prototype.__ntkAppendShieldPatched) {
          Node.prototype.__ntkAppendShieldPatched = true;
          Node.prototype.appendChild = function(node) { return rawAppend.call(this, neuter(node)); };
          Node.prototype.insertBefore = function(node, ref) { return rawInsert.call(this, neuter(node), ref); };
          Element.prototype.prepend = function(...nodes) { return rawPrepend.apply(this, nodes.map(neuter)); };
        }
      } catch (_) {}

      try {
        const obs = new MutationObserver(killDevOverlays);
        obs.observe(document.documentElement, { childList: true, subtree: true });
        window.__ntkDevOverlayObserver = obs;
      } catch (_) {}

      killDevOverlays();
    } catch (_) {}
  }

  installDevtoolsPreflightShield();


  function installAdBlockGuardShield() {
    try {
      if (window.__ntkAdBlockGuardShield) return;
      window.__ntkAdBlockGuardShield = true;

      const BLOCK_IDS = new Set(['ntk_blk_overlay', 'ntk_ad_allow_overlay']);

      function styleTextOf(el) {
        try { return el && el.getAttribute ? (el.getAttribute('style') || '') : ''; } catch (_) { return ''; }
      }

      function textOf(el) {
        try { return el && (el.innerText || el.textContent || '') || ''; } catch (_) { return ''; }
      }

      function looksLikeBlockOverlay(node) {
        try {
          if (!node || node.nodeType !== 1) return false;

          const id = node.id || '';
          const style = styleTextOf(node);
          const text = textOf(node);

          if (BLOCK_IDS.has(id)) return true;
          if (id.includes('ntk_blk')) return true;

          if (text.includes('광고 차단 프로그램이 감지되었습니다')) return true;
          if (text.includes('광고 차단 안내')) return true;
          if (text.includes('광고 차단 확장 프로그램')) return true;
          if (text.includes('도박광고')) return true;

          if (
            style.includes('position: fixed') &&
            style.includes('inset: 0') &&
            style.includes('2147483647') &&
            (
              style.includes('rgba(10, 10, 10') ||
              style.includes('rgba(0, 0, 0') ||
              style.includes('#0a0a0a')
            ) &&
            (
              style.includes('100vw') ||
              style.includes('100vh') ||
              style.includes('display: flex')
            )
          ) return true;

          return false;
        } catch (_) {
          return false;
        }
      }

      function unhide(el) {
        try {
          if (!el || el.nodeType !== 1) return;

          const s = el.style;

          if (s.getPropertyValue('display') === 'none') s.removeProperty('display');
          if (s.getPropertyValue('visibility') === 'hidden') s.removeProperty('visibility');
          if (s.getPropertyValue('opacity') === '0') s.removeProperty('opacity');
          if (s.getPropertyValue('pointer-events') === 'none') s.removeProperty('pointer-events');
          if (s.getPropertyValue('user-select') === 'none') s.removeProperty('user-select');
          if (s.getPropertyValue('-webkit-user-select') === 'none') s.removeProperty('-webkit-user-select');

          if (el.getAttribute('aria-hidden') === 'true') el.removeAttribute('aria-hidden');
          if ('inert' in el && el.inert) el.inert = false;
        } catch (_) {}
      }

      function restoreAdBlockGuardSideEffects() {
        try {
          document.querySelectorAll('#ntk_blk_overlay, #ntk_ad_allow_overlay, [id*="ntk_blk"]').forEach(el => {
            try { el.remove(); } catch (_) {}
          });

          document.querySelectorAll('body > div, body > section, body > aside').forEach(el => {
            if (looksLikeBlockOverlay(el)) {
              try { el.remove(); } catch (_) {}
            }
          });

          if (document.body) {
            Array.from(document.body.children || []).forEach(unhide);
            document.body.style.setProperty('overflow', 'auto', 'important');
            document.body.style.setProperty('pointer-events', 'auto', 'important');
            document.body.style.removeProperty('background');
            document.body.style.removeProperty('background-color');
          }

          document.documentElement.style.setProperty('overflow', 'auto', 'important');
          document.documentElement.style.setProperty('pointer-events', 'auto', 'important');
          document.documentElement.style.removeProperty('background');
          document.documentElement.style.removeProperty('background-color');
        } catch (_) {}
      }

      window.__ntkRestoreAdBlockGuard = restoreAdBlockGuardSideEffects;

      const rawAppendChild = Node.prototype.appendChild;
      const rawInsertBefore = Node.prototype.insertBefore;
      const rawReplaceChild = Node.prototype.replaceChild;
      const rawAppend = Element.prototype.append;
      const rawPrepend = Element.prototype.prepend;
      const rawSetAttribute = Element.prototype.setAttribute;

      function swallow(node) {
        if (looksLikeBlockOverlay(node)) {
          restoreAdBlockGuardSideEffects();
          try {
            node.style.setProperty('display', 'none', 'important');
            node.style.setProperty('visibility', 'hidden', 'important');
            node.style.setProperty('opacity', '0', 'important');
            node.style.setProperty('pointer-events', 'none', 'important');
          } catch (_) {}
          setTimeout(restoreAdBlockGuardSideEffects, 0);
          setTimeout(restoreAdBlockGuardSideEffects, 50);
          setTimeout(restoreAdBlockGuardSideEffects, 200);
          return true;
        }
        return false;
      }

      if (!Node.prototype.__ntkBlockOverlayAppendPatched) {
        Node.prototype.__ntkBlockOverlayAppendPatched = true;

        Node.prototype.appendChild = function(node) {
          if (swallow(node)) return node;
          return rawAppendChild.call(this, node);
        };

        Node.prototype.insertBefore = function(node, ref) {
          if (swallow(node)) return node;
          return rawInsertBefore.call(this, node, ref);
        };

        Node.prototype.replaceChild = function(node, oldNode) {
          if (swallow(node)) return oldNode;
          return rawReplaceChild.call(this, node, oldNode);
        };

        Element.prototype.append = function(...nodes) {
          const safe = nodes.filter(node => !swallow(node));
          if (!safe.length) return;
          return rawAppend.apply(this, safe);
        };

        Element.prototype.prepend = function(...nodes) {
          const safe = nodes.filter(node => !swallow(node));
          if (!safe.length) return;
          return rawPrepend.apply(this, safe);
        };

        Element.prototype.setAttribute = function(name, value) {
          const n = String(name || '').toLowerCase();
          const v = String(value || '');
          if (n === 'id' && BLOCK_IDS.has(v)) {
            restoreAdBlockGuardSideEffects();
            return rawSetAttribute.call(this, 'data-ntk-blocked-id', v);
          }
          return rawSetAttribute.call(this, name, value);
        };
      }

      try {
        const obs = new MutationObserver(() => restoreAdBlockGuardSideEffects());
        obs.observe(document.documentElement, {
          childList: true,
          subtree: true,
          attributes: true,
          attributeFilter: ['style', 'id', 'aria-hidden', 'inert']
        });
        window.__ntkAdBlockGuardObserver = obs;
      } catch (_) {}

      restoreAdBlockGuardSideEffects();
    } catch (_) {}
  }

  installAdBlockGuardShield();

  function earlyCloudflareGuess() {
    const title = document.title || '';
    return (
      title.includes('잠시만') ||
      title.includes('Just a moment') ||
      title.includes('Cloudflare')
    );
  }

  if (earlyCloudflareGuess()) {
    return;
  }

  // closed shadowRoot 선점.
  // 첫 실행 안전모드로 Cloudflare를 먼저 통과한 뒤에만 확장이 로드되므로 여기서 적용.
  try {
    if (!window.__ntkShadowPatched) {
      window.__ntkShadowPatched = true;

      const rawAttachShadow = Element.prototype.attachShadow;
      const shadowStore = new WeakMap();

      Element.prototype.attachShadow = function(init) {
        const opts = Object.assign({}, init || {}, { mode: 'open' });
        const root = rawAttachShadow.call(this, opts);
        shadowStore.set(this, root);
        return root;
      };

      const desc = Object.getOwnPropertyDescriptor(Element.prototype, 'shadowRoot');

      if (desc && desc.get) {
        Object.defineProperty(Element.prototype, 'shadowRoot', {
          configurable: true,
          get() {
            return desc.get.call(this) || shadowStore.get(this) || null;
          }
        });
      }
    }
  } catch (e) {}

  // F12 / 개발자도구 단축키 리다이렉트 방어
  try {
    function isDevtoolsShortcut(e) {
      const key = String(e.key || '').toUpperCase();

      return (
        e.key === 'F12' ||
        e.keyCode === 123 ||
        (
          e.ctrlKey &&
          e.shiftKey &&
          ['I', 'J', 'C'].includes(key)
        ) ||
        (
          e.metaKey &&
          e.altKey &&
          ['I', 'J', 'C'].includes(key)
        )
      );
    }

    function shieldDevKey(e) {
      if (!isDevtoolsShortcut(e)) return;
      e.stopImmediatePropagation();
      e.stopPropagation();
    }

    window.addEventListener('keydown', shieldDevKey, true);
    document.addEventListener('keydown', shieldDevKey, true);
    window.addEventListener('keyup', shieldDevKey, true);
    document.addEventListener('keyup', shieldDevKey, true);
    window.addEventListener('keypress', shieldDevKey, true);
    document.addEventListener('keypress', shieldDevKey, true);

    const rawAddEventListener = EventTarget.prototype.addEventListener;

    EventTarget.prototype.addEventListener = function(type, listener, options) {
      if (
        ['keydown', 'keyup', 'keypress'].includes(type) &&
        typeof listener === 'function'
      ) {
        const wrapped = function(e) {
          if (isDevtoolsShortcut(e)) {
            return;
          }
          return listener.call(this, e);
        };

        return rawAddEventListener.call(this, type, wrapped, options);
      }

      return rawAddEventListener.call(this, type, listener, options);
    };

    ['onkeydown', 'onkeyup', 'onkeypress'].forEach(prop => {
      try {
        Object.defineProperty(window, prop, {
          configurable: true,
          get() {
            return null;
          },
          set() {
            return true;
          }
        });

        Object.defineProperty(document, prop, {
          configurable: true,
          get() {
            return null;
          },
          set() {
            return true;
          }
        });
      } catch (_) {}
    });
  } catch (e) {}

  let antiDebugInstalled = false;

  function installAntiDebug() {
    if (antiDebugInstalled) return;
    antiDebugInstalled = true;

    try {
      const stripDebugger = code => {
        if (typeof code !== 'string') return code;
        return code.replace(/\bdebugger\b\s*;?/g, '');
      };

      const rawEval = window.eval;
      window.eval = function(code) {
        return rawEval.call(this, stripDebugger(code));
      };

      const RawFunction = window.Function;
      window.Function = new Proxy(RawFunction, {
        apply(target, thisArg, args) {
          const cleaned = args.map(arg => stripDebugger(String(arg)));
          return Reflect.apply(target, thisArg, cleaned);
        },
        construct(target, args) {
          const cleaned = args.map(arg => stripDebugger(String(arg)));
          return Reflect.construct(target, cleaned);
        }
      });

      const rawSetTimeout = window.setTimeout;
      window.setTimeout = function(handler, timeout, ...args) {
        if (typeof handler === 'string') {
          handler = stripDebugger(handler);
        } else if (typeof handler === 'function') {
          const src = Function.prototype.toString.call(handler);
          if (src.includes('debugger')) {
            handler = function() {};
          }
        }

        return rawSetTimeout(handler, timeout, ...args);
      };

      const rawSetInterval = window.setInterval;
      window.setInterval = function(handler, timeout, ...args) {
        if (typeof handler === 'string') {
          handler = stripDebugger(handler);
        } else if (typeof handler === 'function') {
          const src = Function.prototype.toString.call(handler);
          if (src.includes('debugger')) {
            handler = function() {};
          }
        }

        return rawSetInterval(handler, timeout, ...args);
      };
    } catch (e) {}
  }

  window.__ntkFullCleaner?.stop?.();

  const MASK_CLASS = '__ntk_ad_mask__';
  const TOOLBAR_ID = '__ntk_toolbar__';
  const BADGE_ID = '__ntk_badge__';
  const AD_STYLE_ID = '__ntk_ad_rules_style__';
  const IP_MODAL_ID = '__ntk_ip_modal__';
  const RANGE_INPUT_ID = '__ntk_range_input__';
  const IP_SERVER = 'http://127.0.0.1:17862/change-ip';

  let lastMaskSignature = '';
  let scheduled = false;

  function isCloudflarePage() {
    const title = document.title || '';
    const text = document.body ? document.body.innerText : '';

    return (
      title.includes('잠시만 기다리') ||
      text.includes('잠시만 기다리') ||
      text.includes('보안 확인 수행 중') ||
      text.includes('Checking if the site connection is secure') ||
      text.includes('Just a moment') ||
      text.includes('Verify you are human') ||
      !!document.querySelector('#challenge-running') ||
      !!document.querySelector('.cf-browser-verification') ||
      !!document.querySelector('iframe[src*="challenges.cloudflare.com"]')
    );
  }

  function shouldDelayPageOps() {
    if (!document.body) return true;

    const text = document.body.innerText || '';
    const title = document.title || '';

    if (document.readyState === 'loading' && text.length < 20 && !title) {
      return true;
    }

    return false;
  }

  function showBadge() {
    if (document.getElementById(BADGE_ID)) return;

    const b = document.createElement('div');
    b.id = BADGE_ID;
    b.textContent = 'NTK ON';
    b.style.cssText = [
      'position:fixed',
      'top:8px',
      'right:8px',
      'z-index:2147483647',
      'background:#00c853',
      'color:#fff',
      'font:12px Arial,sans-serif',
      'padding:4px 8px',
      'border-radius:7px',
      'box-shadow:0 2px 8px rgba(0,0,0,.25)',
      'pointer-events:none'
    ].join(';');

    (document.documentElement || document.body).appendChild(b);

    setTimeout(() => {
      b.remove();
    }, 2500);
  }

  function restoreMouseAndSelection() {
    if (document.body) {
      document.body.style.setProperty('pointer-events', 'auto', 'important');
      document.body.style.setProperty('user-select', 'auto', 'important');
      document.body.style.setProperty('-webkit-user-select', 'auto', 'important');
      document.body.style.setProperty('overflow', 'auto', 'important');

      document.body.onclick = null;
      document.body.oncontextmenu = null;
      document.body.ondragstart = null;
      document.body.onselectstart = null;
    }

    document.documentElement.style.setProperty('pointer-events', 'auto', 'important');
    document.documentElement.style.setProperty('user-select', 'auto', 'important');
    document.documentElement.style.setProperty('-webkit-user-select', 'auto', 'important');
    document.documentElement.style.setProperty('overflow', 'auto', 'important');

    document.oncontextmenu = null;
    document.ondragstart = null;
    document.onselectstart = null;
    window.oncontextmenu = null;
  }

  function hideBadOverlays() {
    const selectors = [
      'div#ntk_blk_overlay',
      'div#ntk_devtools_overlay',
      'div[role="dialog"][aria-modal="true"][style*="z-index: 2147483647"]',
      'div[role="dialog"][aria-modal="true"][style*="z-index: 999999"]',
      'div[role="dialog"][aria-modal="true"][style*="z-index: 999998"]'
    ];

    document.querySelectorAll(selectors.join(',')).forEach(el => {
      try {
        el.remove();
      } catch (_) {
        el.style.setProperty('display', 'none', 'important');
        el.style.setProperty('visibility', 'hidden', 'important');
        el.style.setProperty('opacity', '0', 'important');
        el.style.setProperty('pointer-events', 'none', 'important');
      }
    });

    for (const el of Array.from(document.body?.children || [])) {
      const text = el.textContent || '';
      const style = el.getAttribute('style') || '';

      if (
        text.includes('개발자 도구 차단') ||
        text.includes('접근이 차단되었습니다') ||
        text.includes('광고 차단 프로그램이 감지되었습니다') ||
        text.includes('ntk01@proton.me') ||
        (
          style.includes('position: fixed') &&
          (
            style.includes('2147483647') ||
            style.includes('999999') ||
            style.includes('999998')
          )
        )
      ) {
        try {
          el.remove();
        } catch (_) {
          el.style.setProperty('display', 'none', 'important');
          el.style.setProperty('visibility', 'hidden', 'important');
          el.style.setProperty('opacity', '0', 'important');
          el.style.setProperty('pointer-events', 'none', 'important');
        }
      }
    }

    try { window.__ntkRestoreAdBlockGuard && window.__ntkRestoreAdBlockGuard(); } catch (_) {}
  }

  function applyAdRules() {
    if (window.__ntkAdRulesAppliedStable) return;
    window.__ntkAdRulesAppliedStable = true;

    let style = document.getElementById(AD_STYLE_ID);

    if (!style) {
      style = document.createElement('style');
      style.id = AD_STYLE_ID;
      (document.head || document.documentElement || document.body).appendChild(style);
    }

    const css = `
      [data-br="1"] {
        height: 1px !important;
        min-height: 1px !important;
        max-height: 1px !important;
        overflow: hidden !important;
        margin: 0 !important;
        padding: 0 !important;
      }

      [data-br="1"] > button {
        position: relative !important;
        left: -10000px !important;
        width: 1px !important;
        height: 1px !important;
        overflow: hidden !important;
        display: block !important;
        visibility: visible !important;
        opacity: 1 !important;
      }

      [data-br="1"] img[src*="/board_uploads/"],
      [data-br="1"] img[src*="/api/ad/impression"],
      img[src*="/api/ad/impression"] {
        width: 1px !important;
        height: 1px !important;
        min-width: 1px !important;
        min-height: 1px !important;
        display: block !important;
        visibility: visible !important;
        opacity: 1 !important;
      }

      div#ntk_blk_overlay,
      div#ntk_devtools_overlay,
      div[role="dialog"][aria-modal="true"][style*="z-index: 2147483647"],
      div[role="dialog"][aria-modal="true"][style*="z-index: 999999"],
      div[role="dialog"][aria-modal="true"][style*="z-index: 999998"] {
        display: none !important;
        visibility: hidden !important;
        opacity: 0 !important;
        pointer-events: none !important;
      }

      html,
      body {
        overflow: auto !important;
      }

      header,
      nav,
      header *,
      nav *,
      .nav,
      .nav *,
      .nav-group,
      .nav-group *,
      .header,
      .header * {
        pointer-events: auto !important;
      }
    `;

    if (style.textContent !== css) {
      style.textContent = css;
    }
  }

  function removeOldMasks() {
    document.querySelectorAll('.' + MASK_CLASS).forEach(el => el.remove());
  }

  function isInsideMenuArea(el) {
    if (!el) return false;

    return !!(
      el.closest('header') ||
      el.closest('nav') ||
      el.closest('[role="navigation"]') ||
      el.closest('[class*="header"]') ||
      el.closest('[class*="Header"]') ||
      el.closest('[class*="nav"]') ||
      el.closest('[class*="Nav"]') ||
      el.closest('[class*="menu"]') ||
      el.closest('[class*="Menu"]') ||
      el.closest('[class*="dropdown"]') ||
      el.closest('[class*="Dropdown"]')
    );
  }

  function isExternalLink(a) {
    try {
      if (!a || !a.href) return false;
      const u = new URL(a.href, location.href);
      return u.hostname && u.hostname !== location.hostname;
    } catch {
      return false;
    }
  }

  function getDocRect(el) {
    const r = el.getBoundingClientRect();

    return {
      left: window.scrollX + r.left,
      top: window.scrollY + r.top,
      right: window.scrollX + r.right,
      bottom: window.scrollY + r.bottom,
      width: r.width,
      height: r.height,
      viewTop: r.top,
      viewLeft: r.left
    };
  }

  function isAdImage(img) {
    if (isInsideMenuArea(img)) return false;

    const r = img.getBoundingClientRect();
    if (!r.width || !r.height) return false;

    const ratio = r.width / r.height;
    const parentA = img.closest('a');
    const src = img.src || '';
    const alt = img.alt || '';

    if (r.width < 110 || r.height < 22) return false;

    if (r.height > 260) return false;
    if (ratio < 1.35) return false;

    const bannerShape =
      ratio >= 1.55 &&
      r.width >= 110 &&
      r.height >= 22 &&
      r.height <= 230;

    const nearTop = r.top < 1800;
    const external = isExternalLink(parentA);

    const keyword =
      src.includes('ad') ||
      src.includes('banner') ||
      alt.includes('가입') ||
      alt.includes('카지노') ||
      alt.includes('슬롯') ||
      alt.includes('배팅') ||
      alt.includes('페이백') ||
      alt.includes('가입코드');

    return bannerShape && (nearTop || external || keyword);
  }

  function collectAdRects() {
    const rects = [];

    document.querySelectorAll('img').forEach(img => {
      if (!isAdImage(img)) return;

      const target = img.closest('a') || img;
      if (isInsideMenuArea(target)) return;

      const r = getDocRect(target);

      if (r.width < 80 || r.height < 22) return;

      rects.push(r);
    });

    document.querySelectorAll('iframe').forEach(frame => {
      if (isInsideMenuArea(frame)) return;

      const r = getDocRect(frame);

      if (r.width < 100 || r.height < 30) return;
      if (r.height > 320) return;

      rects.push(r);
    });

    document.querySelectorAll('button[data-bs="1"]').forEach(el => {
      if (isInsideMenuArea(el)) return;

      const r = getDocRect(el);

      if (r.width < 80 || r.height < 25) return;

      rects.push(r);
    });

    return rects;
  }

  function rowClusters(rects) {
    if (!rects.length) return [];

    rects.sort((a, b) => a.top - b.top || a.left - b.left);

    const rows = [];

    for (const r of rects) {
      let row = rows.find(x => Math.abs(x.mid - (r.top + r.height / 2)) < 42);

      if (!row) {
        row = {
          mid: r.top + r.height / 2,
          left: r.left,
          top: r.top,
          right: r.right,
          bottom: r.bottom,
          count: 1
        };
        rows.push(row);
      } else {
        row.left = Math.min(row.left, r.left);
        row.top = Math.min(row.top, r.top);
        row.right = Math.max(row.right, r.right);
        row.bottom = Math.max(row.bottom, r.bottom);
        row.count += 1;
        row.mid = (row.mid + r.top + r.height / 2) / 2;
      }
    }

    return rows
      .map(c => {
        const pad = 4;
        return {
          left: Math.max(0, c.left - pad),
          top: Math.max(0, c.top - pad),
          width: c.right - c.left + pad * 2,
          height: c.bottom - c.top + pad * 2,
          count: c.count
        };
      })
      .filter(c => {
        if (c.width < 120 || c.height < 25) return false;
        if (c.height > 260) return false;
        return true;
      });
  }

  function maskSignature(clusters) {
    return clusters
      .map(c => [
        Math.round(c.left),
        Math.round(c.top),
        Math.round(c.width),
        Math.round(c.height)
      ].join(','))
      .join('|');
  }

  function makeMask(c) {
    const mask = document.createElement('div');
    mask.className = MASK_CLASS;

    mask.style.cssText = [
      'position:absolute',
      `left:${c.left}px`,
      `top:${c.top}px`,
      `width:${c.width}px`,
      `height:${c.height}px`,
      'z-index:999',
      'background:#101010',
      'border-radius:4px',
      'pointer-events:auto',
      'box-sizing:border-box'
    ].join(';');

    mask.onclick = e => {
      e.preventDefault();
      e.stopPropagation();
      e.stopImmediatePropagation();
    };

    mask.onmousedown = e => {
      e.preventDefault();
      e.stopPropagation();
      e.stopImmediatePropagation();
    };

    document.documentElement.appendChild(mask);
  }

  function maskAds(force = false) {
    applyAdRules();
  }

  function getAllRoots() {
    const roots = [document];

    function walkNode(node) {
      if (!node) return;

      let children = [];

      try {
        children = Array.from(node.querySelectorAll('*'));
      } catch (_) {
        return;
      }

      for (const el of children) {
        try {
          if (el.shadowRoot) {
            roots.push(el.shadowRoot);
            walkNode(el.shadowRoot);
          }
        } catch (_) {}
      }
    }

    walkNode(document);
    return roots;
  }

  function cleanText(s) {
    return String(s || '')
      .replace(/\u00a0/g, ' ')
      .replace(/[ \t]+\n/g, '\n')
      .replace(/\n[ \t]+/g, '\n')
      .replace(/[ \t]{2,}/g, ' ')
      .replace(/\n{3,}/g, '\n\n')
      .trim();
  }

  function isBadLine(t) {
    if (!t) return true;

    const bad = [
      'NTK ON',
      '←',
      '→',
      '⟳',
      'TXT',
      '다운로드',
      '로그인',
      '회원가입',
      '광고문의',
      '@newtoki',
      '가입코드',
      '카지노',
      '슬롯',
      '페이백',
      '배팅'
    ];

    return bad.some(x => t.includes(x));
  }

  function extractNovelText() {
    const roots = getAllRoots();

    let best = {
      score: 0,
      lines: []
    };

    for (const root of roots) {
      let candidates = [];

      try {
        candidates = [
          ...Array.from(root.querySelectorAll('.novel-epub-rendered')),
          ...Array.from(root.querySelectorAll('article')),
          ...Array.from(root.querySelectorAll('main')),
          ...Array.from(root.querySelectorAll('[class*="novel"]')),
          root
        ];
      } catch (_) {
        candidates = [root];
      }

      for (const c of candidates) {
        let ps = [];

        try {
          ps = Array.from(c.querySelectorAll('p, .novel-epub-rendered p'));
        } catch (_) {
          ps = [];
        }

        let lines = ps
          .map(p => cleanText(p.textContent))
          .filter(t => t && !isBadLine(t));

        if (lines.length < 3) {
          const raw = cleanText(c.textContent || '');
          const rawLines = raw
            .split(/\n+/)
            .map(cleanText)
            .filter(t => t && !isBadLine(t));

          if (rawLines.length > lines.length) {
            lines = rawLines;
          }
        }

        const totalLen = lines.join('\n').length;
        const score = totalLen + lines.length * 100;

        if (score > best.score) {
          best = {
            score,
            lines
          };
        }
      }
    }

    const text = cleanText(best.lines.join('\n\n'));

    if (!text || text.length < 30) {
      alert('본문을 못 찾았습니다. 소설 페이지에서 새로고침한 뒤 TXT를 다시 눌러보세요.');
      return '';
    }

    return text;
  }

  function safeFileName(name) {
    return String(name || 'novel')
      .replace(/[\\/:*?"<>|]/g, '_')
      .replace(/\s+/g, ' ')
      .trim()
      .slice(0, 120) || 'novel';
  }

  function downloadNovelTxt() {
    const text = extractNovelText();
    if (!text) return;

    const title =
      document.title ||
      document.querySelector('h1, h2, .title, [class*="title"]')?.textContent ||
      'novel';

    const filename = safeFileName(title) + '.txt';

    const blob = new Blob(['\ufeff' + text], {
      type: 'text/plain;charset=utf-8'
    });

    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');

    a.href = url;
    a.download = filename;
    a.style.display = 'none';

    document.documentElement.appendChild(a);
    a.click();

    setTimeout(() => {
      URL.revokeObjectURL(url);
      a.remove();
    }, 1000);
  }


  const BULK_MODAL_ID = '__ntk_bulk_modal__';

  function closeBulkModal() {
    document.getElementById(BULK_MODAL_ID)?.remove();
  }

  function showBulkModal(titleText, subText, showCancel = false) {
    let wrap = document.getElementById(BULK_MODAL_ID);

    if (!wrap) {
      wrap = document.createElement('div');
      wrap.id = BULK_MODAL_ID;
      wrap.style.cssText = [
        'position:fixed',
        'inset:0',
        'z-index:2147483647',
        'display:flex',
        'align-items:center',
        'justify-content:center',
        'background:rgba(9,12,20,.50)',
        'backdrop-filter:blur(8px)',
        'font-family:system-ui,-apple-system,Segoe UI,Roboto,Arial,sans-serif'
      ].join(';');

      const card = document.createElement('div');
      card.id = '__ntk_bulk_card__';
      card.style.cssText = [
        'width:min(420px,calc(100vw - 52px))',
        'padding:30px 26px 24px',
        'border-radius:26px',
        'background:linear-gradient(180deg,rgba(255,255,255,.98),rgba(248,250,252,.96))',
        'box-shadow:0 30px 90px rgba(0,0,0,.34)',
        'border:1px solid rgba(255,255,255,.8)',
        'display:flex',
        'flex-direction:column',
        'align-items:center',
        'gap:14px',
        'color:#111827',
        'text-align:center'
      ].join(';');

      const icon = document.createElement('div');
      icon.style.cssText = [
        'width:58px',
        'height:58px',
        'border-radius:22px',
        'display:grid',
        'place-items:center',
        'background:linear-gradient(135deg,#111827,#334155)',
        'box-shadow:0 16px 35px rgba(17,24,39,.25)'
      ].join(';');

      const spinner = document.createElement('div');
      spinner.style.cssText = [
        'width:28px',
        'height:28px',
        'border-radius:50%',
        'border:3px solid rgba(255,255,255,.28)',
        'border-top-color:#fff',
        'animation:__ntkSpin .85s linear infinite'
      ].join(';');

      const style = document.createElement('style');
      style.textContent = '@keyframes __ntkSpin{to{transform:rotate(360deg)}}';

      const title = document.createElement('div');
      title.id = '__ntk_bulk_title__';
      title.style.cssText = 'font-size:22px;font-weight:900;letter-spacing:-.03em';

      const sub = document.createElement('div');
      sub.id = '__ntk_bulk_sub__';
      sub.style.cssText = 'font-size:14px;color:#64748b;line-height:1.5;white-space:pre-line';

      const cancel = document.createElement('button');
      cancel.id = '__ntk_bulk_cancel__';
      cancel.textContent = '취소';
      cancel.style.cssText = [
        'display:none',
        'margin-top:4px',
        'width:100%',
        'height:42px',
        'border:0',
        'border-radius:14px',
        'background:#e5e7eb',
        'color:#111827',
        'font-weight:800',
        'cursor:pointer'
      ].join(';');

      cancel.addEventListener('click', e => {
        e.preventDefault();
        e.stopPropagation();
        window.__ntkBulkCancel = true;
        cancel.disabled = true;
        cancel.textContent = '취소중...';
      }, true);

      icon.appendChild(spinner);
      card.appendChild(style);
      card.appendChild(icon);
      card.appendChild(title);
      card.appendChild(sub);
      card.appendChild(cancel);
      wrap.appendChild(card);
      document.documentElement.appendChild(wrap);
    }

    const title = document.getElementById('__ntk_bulk_title__');
    const sub = document.getElementById('__ntk_bulk_sub__');
    const cancel = document.getElementById('__ntk_bulk_cancel__');

    if (title) title.textContent = titleText || '저장중입니다';
    if (sub) sub.textContent = subText || '';
    if (cancel) {
      cancel.style.display = showCancel ? 'block' : 'none';
      cancel.disabled = false;
      cancel.textContent = '취소';
    }
  }

  function collectNovelEpisodes() {
    const rows = Array.from(document.querySelectorAll('ul.novel-eps li.novel-ep-row'));

    return rows.map(row => {
      const a = row.querySelector('a.novel-ep-link[href]');
      if (!a) return null;

      const ep = parseInt(row.getAttribute('data-ep') || '0', 10) || 0;
      const num = cleanText(row.querySelector('.ne-num')?.textContent || (ep ? ep + '화' : ''));
      const title = cleanText(row.querySelector('.ne-title')?.textContent || '');

      return {
        ep,
        num,
        title,
        href: new URL(a.getAttribute('href'), location.href).href
      };
    }).filter(Boolean).sort((a, b) => a.ep - b.ep || a.href.localeCompare(b.href));
  }

  function getAllRootsFromDoc(doc) {
    const roots = [doc];

    function walkNode(node) {
      if (!node) return;

      let children = [];

      try {
        children = Array.from(node.querySelectorAll('*'));
      } catch (_) {
        return;
      }

      for (const el of children) {
        try {
          if (el.shadowRoot) {
            roots.push(el.shadowRoot);
            walkNode(el.shadowRoot);
          }
        } catch (_) {}
      }
    }

    walkNode(doc);
    return roots;
  }

  function extractNovelTextFromDoc(doc) {
    const roots = getAllRootsFromDoc(doc);

    let best = {
      score: 0,
      lines: []
    };

    for (const root of roots) {
      let candidates = [];

      try {
        candidates = [
          ...Array.from(root.querySelectorAll('.novel-epub-rendered')),
          ...Array.from(root.querySelectorAll('article')),
          ...Array.from(root.querySelectorAll('main')),
          ...Array.from(root.querySelectorAll('[class*="novel"]')),
          root
        ];
      } catch (_) {
        candidates = [root];
      }

      for (const c of candidates) {
        let ps = [];

        try {
          ps = Array.from(c.querySelectorAll('p, .novel-epub-rendered p'));
        } catch (_) {
          ps = [];
        }

        let lines = ps
          .map(p => cleanText(p.textContent))
          .filter(t => t && !isBadLine(t));

        if (lines.length < 3) {
          const raw = cleanText(c.textContent || '');
          const rawLines = raw
            .split(/\n+/)
            .map(cleanText)
            .filter(t => t && !isBadLine(t));

          if (rawLines.length > lines.length) {
            lines = rawLines;
          }
        }

        const totalLen = lines.join('\n').length;
        const score = totalLen + lines.length * 100;

        if (score > best.score) {
          best = {
            score,
            lines
          };
        }
      }
    }

    return cleanText(best.lines.join('\n\n'));
  }

  function wait(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  const BULK_JOB_KEY = '__ntk_bulk_job_v2__';

  function getBulkJob() {
    try {
      const raw = sessionStorage.getItem(BULK_JOB_KEY);
      return raw ? JSON.parse(raw) : null;
    } catch (_) {
      return null;
    }
  }

  function saveBulkJob(job) {
    try {
      sessionStorage.setItem(BULK_JOB_KEY, JSON.stringify(job));
    } catch (_) {}
  }

  function clearBulkJob() {
    try {
      sessionStorage.removeItem(BULK_JOB_KEY);
    } catch (_) {}
  }

  function currentUrlNoHash() {
    try {
      const u = new URL(location.href);
      u.hash = '';
      return u.href;
    } catch (_) {
      return location.href.split('#')[0];
    }
  }

  function sameEpisodeUrl(a, b) {
    try {
      const ua = new URL(a, location.href);
      const ub = new URL(b, location.href);
      return ua.origin === ub.origin && ua.pathname === ub.pathname;
    } catch (_) {
      return String(a || '').split('#')[0] === String(b || '').split('#')[0];
    }
  }

  function downloadCombinedNovelText(job) {
    const parts = [];
    const failed = [];

    for (const item of job.results || []) {
      const label = item.label || '';

      if (item.text) {
        parts.push(`===== ${label} =====\n\n${item.text}`);
      } else {
        failed.push(`${label}${item.error ? ' - ' + item.error : ''}`);
      }
    }

    if (!parts.length) {
      showBulkModal('저장 실패', failed.length ? failed.join('\n') : '본문을 찾지 못했습니다.', false);
      setTimeout(closeBulkModal, 2600);
      return;
    }

    let text = parts.join('\n\n\n');

    if (failed.length) {
      text += '\n\n\n===== 실패한 회차 =====\n\n' + failed.join('\n');
    }

    const filename = safeFileName((job.title || 'novel') + '_전체') + '.txt';

    const blob = new Blob(['\ufeff' + text], {
      type: 'text/plain;charset=utf-8'
    });

    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');

    a.href = url;
    a.download = filename;
    a.style.display = 'none';

    document.documentElement.appendChild(a);
    a.click();

    setTimeout(() => {
      URL.revokeObjectURL(url);
      a.remove();
    }, 1000);

    const failText = failed.length ? `완료\n실패 ${failed.length}개` : '완료';
    showBulkModal('저장 완료', failText, false);
    setTimeout(closeBulkModal, 1800);
  }

  async function waitCurrentEpisodeText(label, index, total) {
    const started = Date.now();

    while (Date.now() - started < 28000) {
      if (window.__ntkBulkCancel) {
        return { text: '', error: '취소됨' };
      }

      if (isCloudflarePage()) {
        return { text: '', error: 'Cloudflare' };
      }

      showBulkModal('전체 저장중입니다', `${index + 1} / ${total}\n${label}`, true);

      try {
        const text = extractNovelTextFromDoc(document);
        if (text && text.length >= 30) {
          await wait(350);
          const text2 = extractNovelTextFromDoc(document) || text;
          return { text: text2.length >= text.length ? text2 : text, error: '' };
        }
      } catch (_) {}

      await wait(450);
    }

    return { text: '', error: '본문 없음' };
  }

  async function resumeBulkNovelDownload() {
    const job = getBulkJob();
    if (!job || !job.active || !Array.isArray(job.episodes) || !job.episodes.length) return;
    if (window.__ntkBulkRunning) return;

    window.__ntkBulkRunning = true;

    try {
      if (window.__ntkBulkCancel || job.cancelled) {
        clearBulkJob();
        showBulkModal('취소됨', '전체 저장을 취소했습니다.', false);
        setTimeout(closeBulkModal, 1300);
        return;
      }

      const total = job.episodes.length;
      let index = Number(job.index || 0);

      if (index >= total) {
        clearBulkJob();
        downloadCombinedNovelText(job);
        return;
      }

      const ep = job.episodes[index];
      const label = `${ep.num || ep.ep + '화'} ${ep.title || ''}`.trim();

      if (!sameEpisodeUrl(location.href, ep.href)) {
        showBulkModal('전체 저장중입니다', `${index + 1} / ${total}\n${label}`, true);
        location.href = ep.href;
        return;
      }

      const result = await waitCurrentEpisodeText(label, index, total);

      if (window.__ntkBulkCancel || result.error === '취소됨') {
        clearBulkJob();
        showBulkModal('취소됨', '전체 저장을 취소했습니다.', false);
        setTimeout(closeBulkModal, 1300);
        return;
      }

      job.results[index] = {
        label,
        href: ep.href,
        text: result.text || '',
        error: result.error || ''
      };
      job.index = index + 1;
      saveBulkJob(job);

      if (job.index >= total) {
        clearBulkJob();
        downloadCombinedNovelText(job);
        return;
      }

      const next = job.episodes[job.index];
      const nextLabel = `${next.num || next.ep + '화'} ${next.title || ''}`.trim();
      showBulkModal('전체 저장중입니다', `${job.index + 1} / ${total}\n${nextLabel}`, true);

      await wait(350);
      location.href = next.href;
    } finally {
      window.__ntkBulkRunning = false;
    }
  }

  function getRangeInputValue() {
    return cleanText(document.getElementById(RANGE_INPUT_ID)?.value || '');
  }

  function parseEpisodeRange(raw) {
    raw = cleanText(raw || '')
      .replace(/[，,]/g, '~')
      .replace(/[～~−–—-]/g, '~')
      .replace(/화/g, '')
      .trim();

    if (!raw) return null;

    const nums = raw.match(/\d+/g);
    if (!nums || !nums.length) return false;

    let a = parseInt(nums[0], 10);
    let b = nums.length >= 2 ? parseInt(nums[1], 10) : a;

    if (!Number.isFinite(a) || !Number.isFinite(b) || a <= 0 || b <= 0) return false;

    if (a > b) {
      const t = a;
      a = b;
      b = t;
    }

    return { min: a, max: b };
  }

  async function downloadAllNovelTxt() {
    const allEpisodes = collectNovelEpisodes();

    if (!allEpisodes.length) {
      alert('회차 목록을 찾지 못했습니다. 소설 목록 페이지에서 눌러주세요.');
      return;
    }

    const rangeRaw = getRangeInputValue();
    const range = parseEpisodeRange(rangeRaw);

    if (range === false) {
      alert('범위 형식은 3~6 또는 3-6 처럼 입력해주세요.');
      return;
    }

    let episodes = allEpisodes;
    let rangeLabel = '';

    if (range) {
      episodes = allEpisodes.filter(ep => ep.ep >= range.min && ep.ep <= range.max);
      rangeLabel = `${range.min}~${range.max}화`;

      if (!episodes.length) {
        alert(`${rangeLabel}에 해당하는 회차를 찾지 못했습니다.`);
        return;
      }
    }

    const confirmText = range
      ? `${rangeLabel} ${episodes.length}개 회차를 하나의 TXT로 저장할까요?`
      : `총 ${episodes.length}개 회차를 하나의 TXT로 저장할까요?`;

    if (!confirm(confirmText)) {
      return;
    }

    const baseTitle =
      cleanText(document.querySelector('h1, h2, .title, [class*="title"]')?.textContent || '') ||
      cleanText(document.title || '') ||
      'novel';

    const title = range ? `${baseTitle}_${range.min}-${range.max}화` : baseTitle;

    const job = {
      active: true,
      title,
      range,
      listUrl: currentUrlNoHash(),
      index: 0,
      episodes,
      results: new Array(episodes.length).fill(null),
      startedAt: Date.now()
    };

    window.__ntkBulkCancel = false;
    saveBulkJob(job);

    showBulkModal('전체 저장중입니다', `1 / ${episodes.length}\n${episodes[0].num || episodes[0].ep + '화'} ${episodes[0].title || ''}`.trim(), true);

    await wait(350);
    location.href = episodes[0].href;
  }


  function closeIpModal() {
    document.getElementById(IP_MODAL_ID)?.remove();
  }

  function showIpModal(state, message) {
    closeIpModal();

    const wrap = document.createElement('div');
    wrap.id = IP_MODAL_ID;
    wrap.style.cssText = [
      'position:fixed',
      'inset:0',
      'z-index:2147483647',
      'display:flex',
      'align-items:center',
      'justify-content:center',
      'background:rgba(9,12,20,.48)',
      'backdrop-filter:blur(8px)',
      'font-family:system-ui,-apple-system,Segoe UI,Roboto,Arial,sans-serif'
    ].join(';');

    const card = document.createElement('div');
    card.style.cssText = [
      'width:min(380px,calc(100vw - 52px))',
      'padding:30px 26px 26px',
      'border-radius:26px',
      'background:linear-gradient(180deg,rgba(255,255,255,.98),rgba(248,250,252,.96))',
      'box-shadow:0 30px 90px rgba(0,0,0,.34)',
      'border:1px solid rgba(255,255,255,.8)',
      'display:flex',
      'flex-direction:column',
      'align-items:center',
      'gap:14px',
      'color:#111827',
      'text-align:center'
    ].join(';');

    const icon = document.createElement('div');
    icon.style.cssText = [
      'width:58px',
      'height:58px',
      'border-radius:22px',
      'display:grid',
      'place-items:center',
      'background:linear-gradient(135deg,#111827,#334155)',
      'box-shadow:0 16px 35px rgba(17,24,39,.25)'
    ].join(';');

    const spinner = document.createElement('div');
    spinner.style.cssText = [
      'width:28px',
      'height:28px',
      'border-radius:50%',
      'border:3px solid rgba(255,255,255,.28)',
      'border-top-color:#fff',
      'animation:__ntkSpin .85s linear infinite'
    ].join(';');

    const check = document.createElement('div');
    check.textContent = '✓';
    check.style.cssText = [
      'display:none',
      'color:#fff',
      'font-size:30px',
      'font-weight:900',
      'line-height:1'
    ].join(';');

    const style = document.createElement('style');
    style.textContent = '@keyframes __ntkSpin{to{transform:rotate(360deg)}}';

    const title = document.createElement('div');
    title.style.cssText = 'font-size:22px;font-weight:900;letter-spacing:-.03em';
    title.textContent = message || '변경중입니다';

    const sub = document.createElement('div');
    sub.style.cssText = 'font-size:14px;color:#64748b;line-height:1.45';
    sub.textContent = state === 'loading' ? '잠시만 기다려주세요' : '프로그램을 다시 실행해주세요';

    if (state !== 'loading') {
      spinner.style.display = 'none';
      check.style.display = 'block';
    }

    icon.appendChild(spinner);
    icon.appendChild(check);
    card.appendChild(style);
    card.appendChild(icon);
    card.appendChild(title);
    card.appendChild(sub);
    wrap.appendChild(card);

    wrap.addEventListener('click', e => {
      if (e.target !== wrap) return;
      e.preventDefault();
      e.stopPropagation();
    }, false);

    document.documentElement.appendChild(wrap);
  }

  function showIpConfirmModal() {
    closeIpModal();

    return new Promise(resolve => {
      const wrap = document.createElement('div');
      wrap.id = IP_MODAL_ID;
      wrap.style.cssText = [
        'position:fixed',
        'inset:0',
        'z-index:2147483647',
        'display:flex',
        'align-items:center',
        'justify-content:center',
        'background:rgba(9,12,20,.48)',
        'backdrop-filter:blur(8px)',
        'font-family:system-ui,-apple-system,Segoe UI,Roboto,Arial,sans-serif'
      ].join(';');

      const card = document.createElement('div');
      card.style.cssText = [
        'width:min(390px,calc(100vw - 52px))',
        'padding:28px 26px 24px',
        'border-radius:26px',
        'background:linear-gradient(180deg,rgba(255,255,255,.98),rgba(248,250,252,.96))',
        'box-shadow:0 30px 90px rgba(0,0,0,.34)',
        'border:1px solid rgba(255,255,255,.8)',
        'display:flex',
        'flex-direction:column',
        'align-items:center',
        'gap:14px',
        'color:#111827',
        'text-align:center'
      ].join(';');

      const icon = document.createElement('div');
      icon.textContent = 'IP';
      icon.style.cssText = [
        'width:58px',
        'height:58px',
        'border-radius:22px',
        'display:grid',
        'place-items:center',
        'background:linear-gradient(135deg,#111827,#334155)',
        'color:#fff',
        'font-size:18px',
        'font-weight:900',
        'box-shadow:0 16px 35px rgba(17,24,39,.25)'
      ].join(';');

      const title = document.createElement('div');
      title.style.cssText = 'font-size:22px;font-weight:900;letter-spacing:-.03em';
      title.textContent = 'IP 변경이 실행됩니다';

      const sub = document.createElement('div');
      sub.style.cssText = 'font-size:14px;color:#64748b;line-height:1.45';
      sub.textContent = '진행할까요?';

      const actions = document.createElement('div');
      actions.style.cssText = 'display:flex;gap:10px;width:100%;margin-top:4px';

      const no = document.createElement('button');
      no.textContent = '아니오';
      no.type = 'button';
      no.style.cssText = [
        'flex:1',
        'height:42px',
        'border:0',
        'border-radius:14px',
        'background:#e5e7eb',
        'color:#111827',
        'font-weight:800',
        'cursor:pointer'
      ].join(';');

      const yes = document.createElement('button');
      yes.textContent = '예';
      yes.type = 'button';
      yes.style.cssText = [
        'flex:1',
        'height:42px',
        'border:0',
        'border-radius:14px',
        'background:#111827',
        'color:#fff',
        'font-weight:900',
        'cursor:pointer'
      ].join(';');

      function done(v) {
        closeIpModal();
        resolve(v);
      }

      no.addEventListener('click', e => {
        e.preventDefault();
        e.stopPropagation();
        e.stopImmediatePropagation();
        done(false);
      }, true);

      yes.addEventListener('click', e => {
        e.preventDefault();
        e.stopPropagation();
        e.stopImmediatePropagation();
        done(true);
      }, true);

      actions.appendChild(no);
      actions.appendChild(yes);
      card.appendChild(icon);
      card.appendChild(title);
      card.appendChild(sub);
      card.appendChild(actions);
      wrap.appendChild(card);

      wrap.addEventListener('click', e => {
        if (e.target !== wrap) return;
        e.preventDefault();
        e.stopPropagation();
      }, false);

      document.documentElement.appendChild(wrap);
    });
  }

  async function changeIpAddress() {
    const ok = await showIpConfirmModal();
    if (!ok) return;

    showIpModal('loading', '변경중입니다');

    try {
      const res = await fetch(IP_SERVER, {
        method: 'POST',
        cache: 'no-store'
      });

      const data = await res.json().catch(() => ({}));

      if (data && data.ok) {
        showIpModal('done', '다시 실행해주세요');
      } else {
        showIpModal('done', '처리 실패');
      }
    } catch (e) {
      showIpModal('done', '처리 실패');
    }
  }

  function ensureToolbar() {
    if (document.getElementById(TOOLBAR_ID)) return;

    const bar = document.createElement('div');
    bar.id = TOOLBAR_ID;
    bar.style.cssText = [
      'position:fixed',
      'top:8px',
      'left:8px',
      'z-index:2147483647',
      'display:flex',
      'gap:6px',
      'align-items:center',
      'padding:6px',
      'background:rgba(255,255,255,0.94)',
      'border:1px solid rgba(0,0,0,0.2)',
      'border-radius:10px',
      'box-shadow:0 2px 10px rgba(0,0,0,0.15)',
      'font:13px system-ui,-apple-system,Segoe UI,Roboto,Arial'
    ].join(';');

    const btn = (label, onClick) => {
      const b = document.createElement('button');
      b.textContent = label;
      b.type = 'button';
      b.style.cssText = [
        'padding:4px 8px',
        'border:1px solid rgba(0,0,0,0.25)',
        'background:#fff',
        'border-radius:8px',
        'cursor:pointer'
      ].join(';');

      b.addEventListener('click', e => {
        e.preventDefault();
        e.stopPropagation();
        e.stopImmediatePropagation();
        onClick();
      }, true);

      return b;
    };

    const back = btn('←', () => history.back());
    const forward = btn('→', () => history.forward());
    const reload = btn('⟳', () => location.reload());
    const ip = btn('IP', () => changeIpAddress());

    const rangeInput = document.createElement('input');
    rangeInput.id = RANGE_INPUT_ID;
    rangeInput.type = 'text';
    rangeInput.placeholder = '3~6';
    rangeInput.spellcheck = false;
    rangeInput.autocomplete = 'off';
    rangeInput.title = '비워두면 전체 저장, 예: 3~6';
    rangeInput.style.cssText = [
      'width:54px',
      'height:30px',
      'padding:0 8px',
      'border:1px solid rgba(0,0,0,0.25)',
      'background:#fff',
      'border-radius:8px',
      'outline:none',
      'font:13px system-ui,-apple-system,Segoe UI,Roboto,Arial',
      'box-sizing:border-box'
    ].join(';');

    rangeInput.addEventListener('click', e => {
      e.stopPropagation();
    }, true);

    rangeInput.addEventListener('mousedown', e => {
      e.stopPropagation();
    }, true);

    rangeInput.addEventListener('mouseup', e => {
      e.stopPropagation();
    }, true);

    rangeInput.addEventListener('keydown', e => {
      e.stopPropagation();

      if (e.key === 'Enter') {
        e.preventDefault();
        downloadAllNovelTxt();
      }
    }, true);

    rangeInput.addEventListener('keyup', e => {
      e.stopPropagation();
    }, true);

    rangeInput.addEventListener('keypress', e => {
      e.stopPropagation();
    }, true);

    rangeInput.addEventListener('focus', e => {
      e.stopPropagation();
    }, true);

    const bulk = btn('ALL', () => downloadAllNovelTxt());
    const download = btn('TXT', () => downloadNovelTxt());

    bar.appendChild(back);
    bar.appendChild(forward);
    bar.appendChild(reload);
    bar.appendChild(ip);
    bar.appendChild(rangeInput);
    bar.appendChild(bulk);
    bar.appendChild(download);

    let dragging = false;
    let sx = 0;
    let sy = 0;
    let sl = 0;
    let st = 0;

    bar.addEventListener('mousedown', e => {
      const tag = String(e.target?.tagName || '').toUpperCase();
      if (tag === 'BUTTON' || tag === 'INPUT' || tag === 'TEXTAREA' || e.target?.isContentEditable) return;

      dragging = true;
      sx = e.clientX;
      sy = e.clientY;

      const rect = bar.getBoundingClientRect();
      sl = rect.left;
      st = rect.top;

      e.preventDefault();
      e.stopPropagation();
    }, true);

    window.addEventListener('mousemove', e => {
      if (!dragging) return;

      const nx = sl + (e.clientX - sx);
      const ny = st + (e.clientY - sy);

      bar.style.left = Math.max(0, nx) + 'px';
      bar.style.top = Math.max(0, ny) + 'px';
    }, true);

    window.addEventListener('mouseup', () => {
      dragging = false;
    }, true);

    document.documentElement.appendChild(bar);
  }

  window.__ntkDownloadNovelTxt = downloadNovelTxt;
  window.__ntkDownloadAllNovelTxt = downloadAllNovelTxt;


  function installScrollKeeper() {
    try {
      if (window.__ntkScrollKeeperInstalled) return;
      window.__ntkScrollKeeperInstalled = true;

      try { window.__ntkScrollKeeper?.stop?.(); } catch (_) {}

      const state = {
        enabled: true,
        lastY: Math.max(window.scrollY || 0, document.documentElement.scrollTop || 0, document.body?.scrollTop || 0),
        maxH: Math.max(document.documentElement.scrollHeight || 0, document.body?.scrollHeight || 0),
        userUntil: 0,
        href: location.href,
        timer: null,
        observer: null,
        style: null
      };

      function yNow() {
        return Math.max(
          window.scrollY || 0,
          document.documentElement.scrollTop || 0,
          document.body?.scrollTop || 0
        );
      }

      function hNow() {
        return Math.max(
          document.documentElement.scrollHeight || 0,
          document.body?.scrollHeight || 0,
          window.innerHeight || 0
        );
      }

      function userNow() {
        return Date.now() < state.userUntil;
      }

      function markUser(e) {
        try {
          if (e.type === 'wheel') {
            state.userUntil = Date.now() + 700;
            return;
          }

          if (e.type === 'keydown') {
            const k = e.key || '';
            if (['Home', 'PageUp', 'ArrowUp', 'ArrowDown', 'PageDown', 'End', ' '].includes(k)) {
              state.userUntil = Date.now() + 900;
            }
            return;
          }

          state.userUntil = Date.now() + 700;
        } catch (_) {}
      }

      function applyHeightFloor() {
        try {
          const h = state.maxH;

          if (!state.style) {
            state.style = document.createElement('style');
            state.style.id = '__ntk_scroll_keeper_style__';
            document.documentElement.appendChild(state.style);
          }

          state.style.textContent = `
            html, body {
              overflow-anchor: none !important;
              min-height: ${h}px !important;
            }
          `;
        } catch (_) {}
      }

      function resetOnUrlChange() {
        if (location.href === state.href) return;

        state.href = location.href;
        state.lastY = 0;
        state.maxH = hNow();
        applyHeightFloor();
      }

      const rawScrollTo = window.scrollTo.bind(window);
      const rawScroll = window.scroll.bind(window);

      function wantsTop(args) {
        try {
          if (!args || !args.length) return false;

          const a = args[0];

          if (typeof a === 'object' && a) {
            const top = Number(a.top ?? a.y ?? NaN);
            return Number.isFinite(top) && top <= 5;
          }

          if (args.length >= 2) {
            const y = Number(args[1]);
            return Number.isFinite(y) && y <= 5;
          }

          return false;
        } catch (_) {
          return false;
        }
      }

      window.scrollTo = function(...args) {
        if (
          state.enabled &&
          wantsTop(args) &&
          state.lastY > 120 &&
          !userNow()
        ) {
          return;
        }

        return rawScrollTo(...args);
      };

      window.scroll = function(...args) {
        if (
          state.enabled &&
          wantsTop(args) &&
          state.lastY > 120 &&
          !userNow()
        ) {
          return;
        }

        return rawScroll(...args);
      };

      try {
        const rawIntoView = Element.prototype.scrollIntoView;
        Element.prototype.scrollIntoView = function(...args) {
          if (state.enabled && state.lastY > 120 && !userNow()) {
            return;
          }
          return rawIntoView.apply(this, args);
        };
      } catch (_) {}

      try {
        const rawFocus = HTMLElement.prototype.focus;
        HTMLElement.prototype.focus = function(...args) {
          try {
            if (state.enabled && state.lastY > 120 && !userNow()) {
              if (!args.length || typeof args[0] !== 'object') {
                return rawFocus.call(this, { preventScroll: true });
              }
              args[0].preventScroll = true;
            }
          } catch (_) {}

          return rawFocus.apply(this, args);
        };
      } catch (_) {}

      function tick() {
        if (!state.enabled) return;

        resetOnUrlChange();

        const h = hNow();
        if (h > state.maxH) {
          state.maxH = h;
          applyHeightFloor();
        }

        const y = yNow();

        if (!userNow() && state.lastY > 120 && y < 20) {
          rawScrollTo(0, state.lastY);
          return;
        }

        if (y > 20) {
          state.lastY = y;
        }
      }

      ['wheel', 'touchmove', 'mousedown', 'keydown'].forEach(type => {
        window.addEventListener(type, markUser, true);
        document.addEventListener(type, markUser, true);
      });

      state.observer = new MutationObserver(() => {
        const h = hNow();
        if (h > state.maxH) {
          state.maxH = h;
          applyHeightFloor();
        }
      });

      state.observer.observe(document.documentElement, {
        childList: true,
        subtree: true
      });

      applyHeightFloor();

      state.timer = setInterval(tick, 16);

      window.__ntkScrollKeeper = {
        stop() {
          state.enabled = false;
          try { clearInterval(state.timer); } catch (_) {}
          try { state.observer.disconnect(); } catch (_) {}
          try { state.style?.remove(); } catch (_) {}
          console.log('scroll keeper stopped');
        },
        state
      };

      console.log('scroll keeper ON');
    } catch (_) {}
  }

  function installScrollKeeperWhenReady() {
    const ready = () => {
      try {
        if (!document.body) return false;
        if (isCloudflarePage()) return false;
        const textLen = (document.body.innerText || '').length;
        if (document.readyState === 'loading') return false;
        if (textLen < 40 && !document.querySelector('main, header, nav, .container')) return false;
        return true;
      } catch (_) {
        return false;
      }
    };

    const tryInstall = () => {
      if (window.__ntkScrollKeeperInstalled) return;
      if (ready()) installScrollKeeper();
    };

    [1800, 3200, 5200, 8000].forEach(ms => setTimeout(tryInstall, ms));
    window.addEventListener('load', () => setTimeout(tryInstall, 1200), { once: true });
  }

  function clean(force = false) {
    if (shouldDelayPageOps()) return;
    if (isCloudflarePage()) return;

    installAntiDebug();
    restoreMouseAndSelection();
    hideBadOverlays();
    maskAds(force);
    ensureToolbar();
  }

  function bootOnce() {
    if (shouldDelayPageOps()) return;
    if (isCloudflarePage()) return;

    installAntiDebug();
    applyAdRules();
    ensureToolbar();
    restoreMouseAndSelection();
    hideBadOverlays();
  }

  bootOnce();
  installScrollKeeperWhenReady();

  [500, 1400, 3200].forEach(ms => {
    setTimeout(() => {
      try {
        applyAdRules();
        ensureToolbar();
      } catch (_) {}
    }, ms);
  });

  setTimeout(() => {
    resumeBulkNovelDownload();
  }, 900);

  ['contextmenu', 'copy', 'selectstart', 'dragstart'].forEach(type => {
    window.addEventListener(type, e => {
      e.stopPropagation();
    }, true);

    document.addEventListener(type, e => {
      e.stopPropagation();
    }, true);
  });

  function scheduleClean(force = false) {
    if (scheduled) return;
    scheduled = true;

    requestAnimationFrame(() => {
      scheduled = false;
      try {
        hideBadOverlays();
        ensureToolbar();
      } catch (_) {}
    });
  }

  const observer = new MutationObserver(mutations => {
    for (const m of mutations) {
      const nodes = [
        ...Array.from(m.addedNodes || []),
        ...Array.from(m.removedNodes || [])
      ];

      for (const n of nodes) {
        if (!n || n.nodeType !== 1) continue;

        const tag = String(n.tagName || '').toUpperCase();
        const id = String(n.id || '');
        const cls = String(n.className || '');
        const text = String(n.textContent || '');
        const style = String(n.getAttribute?.('style') || '');

        if (tag === 'SCRIPT' || tag === 'STYLE' || tag === 'LINK' || tag === 'META') continue;
        if (id.startsWith('__ntk_') || cls.includes('__ntk_')) continue;

        if (
          id === 'ntk_blk_overlay' ||
          id === 'ntk_devtools_overlay' ||
          text.includes('광고 차단 프로그램이 감지되었습니다') ||
          text.includes('개발자 도구 차단') ||
          (
            style.includes('position: fixed') &&
            (
              style.includes('2147483647') ||
              style.includes('999999') ||
              style.includes('999998')
            )
          )
        ) {
          scheduleClean(false);
          return;
        }
      }
    }
  });

  try {
    observer.observe(document.body || document.documentElement, {
      childList: true,
      subtree: true
    });
  } catch (_) {}

  window.__ntkFullCleaner = {
    stop() {
      try { observer.disconnect(); } catch (_) {}
      document.getElementById(TOOLBAR_ID)?.remove();
      document.getElementById(BADGE_ID)?.remove();
    }
  };

})();
"""

DNR_RULES = [
    {
        "id": 1,
        "priority": 1,
        "action": {"type": "block"},
        "condition": {
            "urlFilter": "/api/dev-block",
            "resourceTypes": ["xmlhttprequest"]
        }
    },
    {
        "id": 2,
        "priority": 1,
        "action": {"type": "block"},
        "condition": {
            "urlFilter": "/api/ev/sync",
            "resourceTypes": ["xmlhttprequest"]
        }
    },
    {
        "id": 3,
        "priority": 1,
        "action": {"type": "block"},
        "condition": {
            "urlFilter": "/api/ev/etag",
            "resourceTypes": ["xmlhttprequest"]
        }
    },
    {
        "id": 10,
        "priority": 2,
        "action": {"type": "allow"},
        "condition": {
            "urlFilter": "/api/ad/challenge",
            "resourceTypes": ["xmlhttprequest"]
        }
    },
    {
        "id": 11,
        "priority": 2,
        "action": {"type": "allow"},
        "condition": {
            "urlFilter": "/api/ad/ack",
            "resourceTypes": ["xmlhttprequest"]
        }
    },
    {
        "id": 12,
        "priority": 2,
        "action": {"type": "allow"},
        "condition": {
            "urlFilter": "/api/ad/impression",
            "resourceTypes": ["xmlhttprequest", "image"]
        }
    }
]

MANIFEST = {
    "manifest_version": 3,
    "name": "NTK Full Cleaner",
    "version": "1.8.3",
    "description": "Local page cleaner",
    "permissions": ["declarativeNetRequest"],
    "declarative_net_request": {
        "rule_resources": [
            {
                "id": "ruleset_1",
                "enabled": True,
                "path": "rules.json"
            }
        ]
    },
    "host_permissions": [
        "https://sbxh2.com/*",
        "https://*.sbxh2.com/*",
        "https://whoas.xyz/*",
        "https://*.whoas.xyz/*",
        "http://127.0.0.1:17862/*",
        "http://localhost:17862/*",
        "https://i.toonflix.app/*"
    ],
    "content_scripts": [
        {
            "matches": [
                "https://sbxh2.com/*",
                "https://*.sbxh2.com/*",
                "https://whoas.xyz/*",
                "https://*.whoas.xyz/*"
            ],
            "js": ["content.js"],
            "run_at": "document_start",
            "all_frames": True,
            "world": "MAIN"
        }
    ]
}

def ask_yes_no(title, message):
    try:
        import tkinter as tk
        from tkinter import messagebox

        root = tk.Tk()
        root.withdraw()
        result = messagebox.askyesno(title, message)
        root.destroy()
        return result
    except Exception:
        print(f"[{title}] {message}")
        answer = input("성공했으면 y, 아니면 n 입력: ").strip().lower()
        return answer == "y"


def info_box(title, message):
    try:
        import tkinter as tk
        from tkinter import messagebox

        root = tk.Tk()
        root.withdraw()
        messagebox.showinfo(title, message)
        root.destroy()
    except Exception:
        print(f"[{title}] {message}")


def find_edge():
    for path in EDGE_PATHS:
        if os.path.exists(path):
            return path
    raise FileNotFoundError("Edge 경로를 못 찾았습니다.")


def make_extension():
    EXT_DIR.mkdir(parents=True, exist_ok=True)

    old_bg = EXT_DIR / "background.js"
    if old_bg.exists():
        old_bg.unlink()

    (EXT_DIR / "manifest.json").write_text(
        json.dumps(MANIFEST, ensure_ascii=False, indent=2),
        encoding="utf-8"
    )

    (EXT_DIR / "content.js").write_text(
        CONTENT_JS,
        encoding="utf-8"
    )

    (EXT_DIR / "rules.json").write_text(
        json.dumps(DNR_RULES, ensure_ascii=False, indent=2),
        encoding="utf-8"
    )


def reset_profile():
    if PROFILE_DIR.exists():
        shutil.rmtree(PROFILE_DIR, ignore_errors=True)

    if PASS_FLAG.exists():
        PASS_FLAG.unlink()

    info_box(
        "초기화 완료",
        "전용 프로필을 초기화했습니다.\n다시 실행하면 안전모드로 열립니다."
    )


def kill_edge_once():
    try:
        subprocess.run(
            ["taskkill", "/F", "/IM", "msedge.exe"],
            stdout=subprocess.DEVNULL,
            stderr=subprocess.DEVNULL,
            check=False
        )
    except Exception:
        pass


def launch_edge(load_extension: bool):
    # 실행 전에 Edge 전부 종료
    # 주의: 일반 Edge 창도 같이 닫힘
    kill_edge_once()

    edge = find_edge()
    PROFILE_DIR.mkdir(parents=True, exist_ok=True)

    args = [
        edge,

        # 전용 프로필
        f"--user-data-dir={PROFILE_DIR}",

        # Edge 백그라운드 잔류 방지
        "--disable-background-mode",
        "--no-first-run",
        "--no-default-browser-check",
        "--disable-features=msEdgeStartupBoost,StartupBoost",

        # 앱창
        f"--app={URL}",
    ]

    if load_extension:
        make_extension()

        # 확장 강제 로드
        args.insert(2, f"--disable-extensions-except={EXT_DIR}")
        args.insert(3, f"--load-extension={EXT_DIR}")

    return subprocess.Popen(args)


def center_tk_window(win, w, h):
    try:
        sw = win.winfo_screenwidth()
        sh = win.winfo_screenheight()
        x = int((sw - w) / 2)
        y = int((sh - h) / 2)
        win.geometry(f"{w}x{h}+{x}+{y}")
    except Exception:
        win.geometry(f"{w}x{h}")


def native_reset_and_exit():
    try:
        import tkinter as tk
        from tkinter import ttk

        root = tk.Tk()
        root.title("IP")
        root.resizable(False, False)
        root.attributes("-topmost", True)
        center_tk_window(root, 420, 240)

        outer = tk.Frame(root, bg="#0f172a")
        outer.pack(fill="both", expand=True)

        card = tk.Frame(outer, bg="#ffffff", bd=0, highlightthickness=1, highlightbackground="#e5e7eb")
        card.place(relx=0.5, rely=0.5, anchor="center", width=360, height=180)

        title = tk.Label(card, text="변경중입니다", bg="#ffffff", fg="#111827", font=("맑은 고딕", 17, "bold"))
        title.pack(pady=(30, 8))

        sub = tk.Label(card, text="잠시만 기다려주세요", bg="#ffffff", fg="#64748b", font=("맑은 고딕", 10))
        sub.pack()

        bar = ttk.Progressbar(card, mode="indeterminate", length=250)
        bar.pack(pady=(22, 0))
        bar.start(12)

        def finish(ok=True):
            try:
                bar.stop()
            except Exception:
                pass
            title.config(text="다시 실행해주세요" if ok else "처리 실패")
            sub.config(text="프로그램을 종료합니다" if ok else "창을 닫고 다시 시도해주세요")
            root.after(1600 if ok else 2400, lambda: os._exit(0) if ok else root.destroy())

        def worker():
            result = run_ip_helper_elevated()
            reset_extension_flag()
            try:
                kill_edge_once()
            except Exception:
                pass
            root.after(0, lambda: finish(bool(result.get("ok"))))

        threading.Thread(target=worker, daemon=True).start()
        root.mainloop()
    except Exception:
        try:
            run_ip_helper_elevated()
        except Exception:
            pass
        reset_extension_flag()
        try:
            kill_edge_once()
        except Exception:
            pass
        os._exit(0)


def first_run_choice_dialog():
    try:
        import tkinter as tk

        result = {"value": "no"}

        root = tk.Tk()
        root.title("첫 실행 안전모드")
        root.resizable(False, False)
        root.attributes("-topmost", True)
        center_tk_window(root, 560, 325)

        bg = "#f8fafc"
        root.configure(bg=bg)

        card = tk.Frame(root, bg="#ffffff", highlightthickness=1, highlightbackground="#e5e7eb")
        card.place(relx=0.5, rely=0.5, anchor="center", width=515, height=275)

        icon = tk.Label(card, text="?", bg="#2563eb", fg="#ffffff", font=("맑은 고딕", 18, "bold"), width=2, height=1)
        icon.place(x=24, y=28, width=42, height=42)

        title = tk.Label(card, text="첫 실행 안전모드", bg="#ffffff", fg="#111827", font=("맑은 고딕", 15, "bold"))
        title.place(x=82, y=28)

        msg = (
            "잠시뒤 짭토끼 사이트나 클라우드플레어 인증창이 뜨면\n"
            "사람인증 해주시고 예를 눌러주세요.\n\n"
            "클라우드플레어 또는 연결 오류 화면이면 아니오를 누르세요.\n"
            "1006 에러가 뜨면 IP 버튼을 눌러주세요.\n\n"
            "[예]를 누르면 다음 실행부터 기능을 켭니다."
        )

        body = tk.Label(card, text=msg, justify="left", bg="#ffffff", fg="#334155", font=("맑은 고딕", 10), anchor="w")
        body.place(x=82, y=66)

        btn_frame = tk.Frame(card, bg="#ffffff")
        btn_frame.place(x=24, y=215, width=467, height=42)

        def choose(v):
            result["value"] = v
            root.destroy()

        def make_btn(text, command, bgc, fgc="#111827", w=92):
            b = tk.Button(
                btn_frame,
                text=text,
                command=command,
                bg=bgc,
                fg=fgc,
                activebackground=bgc,
                activeforeground=fgc,
                relief="flat",
                bd=0,
                font=("맑은 고딕", 10, "bold"),
                cursor="hand2"
            )
            b.pack(side="right", padx=(8, 0), ipadx=12, ipady=7)
            return b

        make_btn("예", lambda: choose("yes"), "#2563eb", "#ffffff")
        make_btn("아니오", lambda: choose("no"), "#e5e7eb", "#111827")
        make_btn("IP", lambda: choose("ip"), "#111827", "#ffffff")

        root.protocol("WM_DELETE_WINDOW", lambda: choose("no"))
        root.mainloop()
        return result["value"]
    except Exception:
        return "yes" if ask_yes_no(
            "첫 실행 안전모드",
            "잠시뒤 짭토끼 사이트나 클라우드플레어 인증창이 뜨면 사람인증 해주시고 예를 눌러주세요.\n\n"
            "클라우드플레어 또는 연결 오류 화면이면 아니오를 누르세요.\n"
            "1006 에러가 뜨면 IP 버튼을 눌러주세요.\n\n"
            "[예]를 누르면 다음 실행부터 기능을 켭니다."
        ) else "no"


def first_run_safe_mode():
    launch_edge(load_extension=False)

    choice = first_run_choice_dialog()

    if choice == "ip":
        try:
            import tkinter as tk
            from tkinter import messagebox

            root = tk.Tk()
            root.withdraw()
            root.attributes("-topmost", True)
            ok = messagebox.askyesno("IP 변경", "IP 변경이 실행됩니다.\n진행할까요?", parent=root)
            root.destroy()
        except Exception:
            ok = True

        if ok:
            native_reset_and_exit()
        return

    if choice == "yes":
        PASS_FLAG.write_text("cloudflare/site passed\n", encoding="utf-8")
        info_box(
            "설정 완료",
            "다음 실행부터 기능을 켭니다.\n"
            "지금 열린 창을 닫고 프로그램을 다시 실행하세요."
        )
    else:
        if PASS_FLAG.exists():
            PASS_FLAG.unlink()

        info_box(
            "안전모드 유지",
            "아직 통과 성공으로 저장하지 않았습니다.\n"
            "다시 실행하면 또 안전모드로 열립니다."
        )


def main():
    BASE_DIR.mkdir(parents=True, exist_ok=True)

    if "--ip-helper" in sys.argv:
        idx = sys.argv.index("--ip-helper")
        result_path = sys.argv[idx + 1] if len(sys.argv) > idx + 1 else str(BASE_DIR / "ip_change_result.json")
        run_ip_helper(result_path)
        return

    if "--reset" in sys.argv:
        reset_profile()
        return

    if "--safe" in sys.argv:
        launch_edge(load_extension=False)
        return

    if "--enable" in sys.argv:
        PASS_FLAG.write_text("manual enable\n", encoding="utf-8")
        start_ip_server()
        launch_edge(load_extension=True)
        wait_for_edge_exit()
        return

    if "--disable" in sys.argv:
        if PASS_FLAG.exists():
            PASS_FLAG.unlink()
        launch_edge(load_extension=False)
        return

    if not PASS_FLAG.exists():
        first_run_safe_mode()
        return

    start_ip_server()
    launch_edge(load_extension=True)
    wait_for_edge_exit()


if __name__ == "__main__":
    main()

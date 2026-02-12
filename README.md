#!/bin/bash
###############################################################################
# Pi Zero 2 W + VIOFO A329S + SIM7600G-H HAT (B) 최종 셋업
#
# 사전 준비:
#   1. Raspberry Pi Imager로 Lite 굽기 (SSH + 집 Wi-Fi 설정)
#   2. SIM7600 HAT에 유심+안테나, Pi에 얹기 (pogo pin 접촉!)
#   3. Pi 전원 → SSH 접속 → 이 스크립트 실행
#
# 사용법:
#   chmod +x setup.sh
#   sudo ./setup.sh       # 1단계: config 변경 + 재부팅
#   sudo ./setup.sh       # 2단계: 설치 완료
#
# 구조:
#   [A329S] ──Wi-Fi(2.4G)──> [Pi Zero 2 W + SIM7600] ──LTE──> Tailscale
#    (AP)       wlan0             USB HAT                         │
#                                                           폰에서 원격 접속
###############################################################################

set -e

# ============================================================
# 설정값 (여기만 수정!)
# ============================================================
DASHCAM_SSID="A329"                # 대시캠 Wi-Fi SSID
DASHCAM_PASS="Shqk1379"           # 대시캠 Wi-Fi 비번
DASHCAM_IP="192.168.1.254"        # VIOFO 기본값
LTE_APN="lte.ktfwing.com"         # KT (SKT: lte.sktelecom.com, LGU+: lte.lguplus.co.kr)

PHASE_FILE="/home/$SUDO_USER/.setup_phase1_done"

# ============================================================
# 1단계: 부팅 설정 (재부팅 필요)
# ============================================================
phase1() {
    echo "========================================"
    echo "  1단계: 부팅 설정"
    echo "========================================"

    BOOT="/boot/firmware"
    [ ! -d "$BOOT" ] && BOOT="/boot"

    # --- config.txt: dwc2 host 모드 (HAT USB 인식용) ---
    if grep -q "^dtoverlay=dwc2$" "$BOOT/config.txt"; then
        sed -i 's/^dtoverlay=dwc2$/dtoverlay=dwc2,dr_mode=host/' "$BOOT/config.txt"
    elif grep -q "dtoverlay=dwc2" "$BOOT/config.txt"; then
        sed -i 's/dtoverlay=dwc2.*/dtoverlay=dwc2,dr_mode=host/' "$BOOT/config.txt"
    else
        echo "dtoverlay=dwc2,dr_mode=host" >> "$BOOT/config.txt"
    fi
    echo "✅ dwc2 host 모드"

    if ! grep -q "enable_uart=1" "$BOOT/config.txt"; then
        echo "enable_uart=1" >> "$BOOT/config.txt"
    fi
    echo "✅ UART 활성화"

    # --- cmdline.txt: gadget 모듈 제거 (HAT USB와 충돌) ---
    sed -i 's/ modules-load=dwc2[^ ]*//' "$BOOT/cmdline.txt"
    echo "✅ gadget 모듈 제거"

    # --- 시리얼 콘솔 비활성화 ---
    raspi-config nonint do_serial_cons 1
    echo "✅ 시리얼 콘솔 비활성화"

    touch "$PHASE_FILE"
    echo ""
    echo "  1단계 완료! 재부팅합니다."
    echo "  재부팅 후 다시: sudo ./setup.sh"
    sleep 3
    reboot
}

# ============================================================
# 2단계: 소프트웨어 설치 + 설정
# ============================================================
phase2() {
    echo "========================================"
    echo "  2단계: 소프트웨어 설치"
    echo "========================================"

    # --- 패키지 (집 Wi-Fi로 다운로드) ---
    echo "📦 패키지 설치..."
    apt update
    apt install -y ffmpeg nginx fcgiwrap

    # --- Tailscale (집 Wi-Fi로 다운로드) ---
    echo ""
    echo "🔒 Tailscale 설치..."
    if ! command -v tailscale &>/dev/null; then
        curl -fsSL https://tailscale.com/install.sh | sh
    fi
    echo "✅ Tailscale 설치 완료"

    # ======================================================
    # LTE
    # ======================================================
    echo ""
    echo "📱 LTE 설정..."

    if ! lsusb | grep -qi "simtech\|qualcomm"; then
        echo "⚠️ SIM7600 USB 안 보임. HAT pogo pin 접촉 확인!"
        exit 1
    fi
    echo "✅ SIM7600 인식됨"

    systemctl enable ModemManager
    systemctl start ModemManager
    sleep 5

    if ! mmcli -L 2>/dev/null | grep -qi "SIM\|Qualcomm"; then
        echo "⏳ 모뎀 대기..."
        sleep 10
    fi

    nmcli con delete lte 2>/dev/null || true
    nmcli con add type gsm ifname '*' con-name lte apn "$LTE_APN"
    nmcli con modify lte connection.autoconnect yes

    echo "⏳ LTE 연결..."
    if nmcli con up lte; then
        echo "✅ LTE 연결 성공!"
    else
        echo "⚠️ LTE 실패 — 나중에: sudo nmcli con up lte"
    fi

    # --- Tailscale 로그인 ---
    echo ""
    echo "🔒 Tailscale 로그인..."
    tailscale up
    TS_IP=$(tailscale ip -4 2>/dev/null || echo "확인안됨")
    echo "✅ Tailscale IP: $TS_IP"

    # ======================================================
    # 대시캠 Wi-Fi
    # ======================================================
    echo ""
    echo "📷 대시캠 Wi-Fi 설정..."
    nmcli con delete dashcam 2>/dev/null || true
    nmcli con add type wifi ifname wlan0 con-name dashcam \
        ssid "$DASHCAM_SSID"
    nmcli con modify dashcam \
        wifi-sec.key-mgmt wpa-psk \
        wifi-sec.psk "$DASHCAM_PASS" \
        ipv4.never-default yes \
        connection.autoconnect yes \
        connection.autoconnect-priority 20
    echo "✅ 대시캠 Wi-Fi 프로필 생성"

    # 대시캠 외 Wi-Fi 삭제 (집 Wi-Fi 등)
    nmcli -t -f NAME,TYPE con show | grep ':.*wireless' | cut -d: -f1 | while read name; do
        if [ "$name" != "dashcam" ]; then
            nmcli con delete "$name" 2>/dev/null
            echo "🗑️ Wi-Fi 삭제: $name"
        fi
    done

    # ======================================================
    # RTSP → HLS 릴레이
    # ======================================================
    echo ""
    echo "📡 스트리밍 서버..."

    mkdir -p /var/www/html/hls
    chmod 777 /var/www/html/hls

    cat > /usr/local/bin/rtsp-relay.sh << 'RELAY_EOF'
#!/bin/bash
DASHCAM_IP="192.168.1.254"
RTSP_URL="rtsp://${DASHCAM_IP}:554"
HLS_DIR="/var/www/html/hls"
PID_FILE="/var/run/rtsp-relay.pid"

case "$1" in
    start)
        if [ -f "$PID_FILE" ] && kill -0 "$(cat $PID_FILE)" 2>/dev/null; then
            echo "running"; exit 0
        fi
        if ! ping -c 1 -W 2 "$DASHCAM_IP" &>/dev/null; then
            echo "dashcam_offline"; exit 1
        fi
        mkdir -p "$HLS_DIR"
        rm -f "$HLS_DIR"/live_*.ts "$HLS_DIR/live.m3u8"
        ffmpeg -fflags +genpts+discardcorrupt \
            -rtsp_transport tcp \
            -i "$RTSP_URL" \
            -c:v copy -an \
            -bsf:v h264_mp4toannexb \
            -f hls -hls_time 4 -hls_list_size 5 \
            -hls_flags delete_segments+append_list \
            -hls_segment_filename "$HLS_DIR/live_%04d.ts" \
            "$HLS_DIR/live.m3u8" </dev/null >/dev/null 2>&1 &
        echo $! > "$PID_FILE"
        echo "started"
        ;;
    stop)
        [ -f "$PID_FILE" ] && kill "$(cat $PID_FILE)" 2>/dev/null
        rm -f "$PID_FILE" "$HLS_DIR"/live_*.ts "$HLS_DIR/live.m3u8"
        echo "stopped"
        ;;
    status)
        if [ -f "$PID_FILE" ] && kill -0 "$(cat $PID_FILE)" 2>/dev/null; then
            echo "running"
        else
            echo "stopped"
        fi
        ;;
esac
RELAY_EOF
    chmod +x /usr/local/bin/rtsp-relay.sh

    # ======================================================
    # 워치독 (2분 미시청 자동 중지)
    # ======================================================
    cat > /usr/local/bin/relay-watchdog.sh << 'WD_EOF'
#!/bin/bash
TIMEOUT=120
IDLE=0
while true; do
    sleep 30
    PID_FILE="/var/run/rtsp-relay.pid"
    [ ! -f "$PID_FILE" ] && IDLE=0 && continue
    kill -0 "$(cat $PID_FILE)" 2>/dev/null || { IDLE=0; continue; }
    HIT=$(find /var/log/nginx/ -name "access.log" -newer "$PID_FILE" \
        -exec grep -cl "live" {} + 2>/dev/null | head -1)
    if [ -n "$HIT" ]; then IDLE=0; else
        IDLE=$((IDLE + 30))
        [ "$IDLE" -ge "$TIMEOUT" ] && /usr/local/bin/rtsp-relay.sh stop && IDLE=0
    fi
done
WD_EOF
    chmod +x /usr/local/bin/relay-watchdog.sh

    cat > /etc/systemd/system/relay-watchdog.service << 'SVC_EOF'
[Unit]
Description=RTSP Relay Watchdog
After=nginx.service
[Service]
Type=simple
ExecStart=/usr/local/bin/relay-watchdog.sh
Restart=always
[Install]
WantedBy=multi-user.target
SVC_EOF

    # ======================================================
    # CGI (웹 버튼 → 릴레이 제어)
    # ======================================================
    # sudo 권한 부여
    echo "www-data ALL=(ALL) NOPASSWD: /usr/local/bin/rtsp-relay.sh" > /etc/sudoers.d/rtsp-relay
    chmod 440 /etc/sudoers.d/rtsp-relay

    mkdir -p /usr/lib/cgi-bin
    for cmd in start stop status; do
        cat > /usr/lib/cgi-bin/relay-${cmd}.sh << CGISCRIPT
#!/bin/bash
echo "Content-Type: text/plain"
echo ""
sudo /usr/local/bin/rtsp-relay.sh ${cmd}
CGISCRIPT
        chmod +x /usr/lib/cgi-bin/relay-${cmd}.sh
    done

    # ======================================================
    # Nginx
    # ======================================================
    cat > /etc/nginx/sites-available/dashcam << 'NGINX_EOF'
server {
    listen 8080;

    location /hls/ {
        alias /var/www/html/hls/;
        types {
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
        }
        add_header Cache-Control no-cache;
        add_header Access-Control-Allow-Origin *;
    }

    location ~ ^/api/(start|stop|status)$ {
        fastcgi_pass unix:/var/run/fcgiwrap.socket;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME /usr/lib/cgi-bin/relay-$1.sh;
    }

    location /dashcam/ { proxy_pass http://192.168.1.254/; }
    location /DCIM/ { proxy_pass http://192.168.1.254/DCIM/; }

    location / { root /var/www/html; index player.html; }
}
NGINX_EOF

    ln -sf /etc/nginx/sites-available/dashcam /etc/nginx/sites-enabled/
    rm -f /etc/nginx/sites-enabled/default

    # ======================================================
    # 웹 UI
    # ======================================================
    cat > /var/www/html/player.html << 'HTML_EOF'
<!DOCTYPE html>
<html lang="ko">
<head>
<title>🚗 대시캠</title>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<script src="https://cdnjs.cloudflare.com/ajax/libs/hls.js/1.4.12/hls.min.js"></script>
<style>
*{box-sizing:border-box;margin:0}
body{background:#0a0a0a;color:#e0e0e0;font-family:system-ui;
     padding:16px;display:flex;flex-direction:column;align-items:center;max-width:900px;margin:0 auto}
h1{font-size:1.3em;margin-bottom:8px}
video{width:100%;border-radius:8px;background:#000;margin:8px 0}
.row{display:flex;gap:6px;margin:6px 0;flex-wrap:wrap;justify-content:center}
.b{padding:8px 16px;border-radius:8px;border:none;font-size:.85em;
   cursor:pointer;font-weight:600;transition:.2s}
.b1{background:#2563eb;color:#fff}.b1:hover{background:#1d4ed8}
.b2{background:#444;color:#ddd}.b2:hover{background:#555}
.b3{background:#222;color:#aaa;border:1px solid #444}.b3:hover{background:#333}
.b.active{outline:2px solid #4ade80;outline-offset:2px}
.st{font-size:.85em;padding:6px 14px;border-radius:6px;margin:6px 0}
.on{background:#14532d;color:#4ade80}
.off{background:#1c1917;color:#a8a29e}
.ld{background:#422006;color:#fbbf24}
.label{font-size:.7em;color:#666;margin-top:8px;letter-spacing:1px}
.info{font-size:.75em;color:#555;margin-top:12px;text-align:center}
#msg{font-size:.8em;color:#888;margin:4px 0;min-height:1.2em}
#photo{display:none;margin:8px 0;text-align:center}
#photo img{max-width:100%;border-radius:8px;margin-bottom:6px}
</style>
</head>
<body>
<h1>🚗 A329S</h1>
<video id="v" controls autoplay muted playsinline></video>
<div class="st off" id="st">대기 중</div>
<div id="msg"></div>
<div id="photo"></div>

<div class="label">스트리밍</div>
<div class="row">
 <button class="b b1" onclick="go()">▶ 시작</button>
 <button class="b b2" onclick="stop()">⏹ 중지</button>
</div>

<div class="label">카메라</div>
<div class="row">
 <button class="b b3" id="cam0" onclick="switchCam(0)">📷 전방</button>
 <button class="b b3" id="cam1" onclick="switchCam(1)">📷 후방</button>
</div>

<div class="label">촬영 (고해상도)</div>
<div class="row">
 <button class="b b3" onclick="takePhoto(0)">📸 전방 촬영</button>
 <button class="b b3" onclick="takePhoto(1)">📸 후방 촬영</button>
</div>

<div class="label">녹화</div>
<div class="row">
 <button class="b b3" onclick="cam('2001','1')">⏺ 녹화 시작</button>
 <button class="b b3" onclick="cam('2001','0')">⏹ 녹화 중지</button>
</div>

<div class="label">기타</div>
<div class="row">
 <button class="b b3" onclick="getInfo()">ℹ️ 정보</button>
 <button class="b b3" onclick="window.open('/dashcam/')">📁 파일</button>
</div>

<div class="info">2분 미시청 시 자동 중지 · LTE 데이터 ~10MB/분</div>

<script>
const v=document.getElementById('v'),s=document.getElementById('st'),mg=document.getElementById('msg');
const ph=document.getElementById('photo');
let h,curCam=0;
function ss(c,m){s.className='st '+c;s.textContent=m}
function msg(m){mg.textContent=m;setTimeout(()=>{if(mg.textContent===m)mg.textContent=''},5000)}

async function api(c){try{return(await(await fetch('/api/'+c)).text()).trim()}catch{return'error'}}

async function cam(cmd,par){
  let url='/dashcam/?custom=1&cmd='+cmd;
  if(par!==undefined)url+='&par='+par;
  try{
    const r=await fetch(url);
    const t=await r.text();
    const st=t.match(/<Status>(-?\d+)<\/Status>/);
    msg(st&&st[1]==='0'?'✅ 성공':'⚠️ 응답: '+(st?st[1]:'알수없음'));
    return st?st[1]:'error';
  }catch(e){msg('❌ 통신 실패');return'error'}
}

async function takePhoto(camNum){
  const label=camNum===0?'전방':'후방';
  msg('📸 '+label+' 촬영 중...');
  await cam('3028',String(camNum));
  await new Promise(r=>setTimeout(r,1000));
  const r=await cam('2017');
  if(r!=='0'){msg('❌ 촬영 실패');return}
  await cam('3028',String(curCam));
  msg('📸 촬영 완료! 파일 찾는 중...');
  await new Promise(r=>setTimeout(r,2000));
  try{
    const paths=['/dashcam/DCIM/Photo/','/dashcam/DCIM/'];
    for(const base of paths){if(await findPhoto(base))return}
    msg('⚠️ 📁 파일에서 확인하세요');
  }catch(e){msg('⚠️ 📁 파일에서 확인하세요')}
}

async function findPhoto(base){
  try{
    const res=await fetch(base);
    if(!res.ok)return false;
    const html=await res.text();
    const folders=[...html.matchAll(/href="([^"]*\/)"/g)].map(m=>m[1]);
    const jpgs=[...html.matchAll(/href="([^"]*\.(?:jpg|jpeg|JPG|JPEG))"/gi)].map(m=>m[1]);
    if(jpgs.length>0){
      const last=jpgs[jpgs.length-1];
      const url=last.startsWith('/')?'/dashcam'+last:base+last;
      showPhoto(url);return true;
    }
    for(const f of folders.reverse()){
      const fres=await fetch(base+f);
      if(!fres.ok)continue;
      const fhtml=await fres.text();
      const imgs=[...fhtml.matchAll(/href="([^"]*\.(?:jpg|jpeg|JPG|JPEG))"/gi)].map(m=>m[1]);
      if(imgs.length>0){
        const u=imgs[imgs.length-1];
        const url=u.startsWith('/')?'/dashcam'+u:base+f+u;
        showPhoto(url);return true;
      }
    }
  }catch(e){}
  return false;
}

function showPhoto(url){
  ph.style.display='block';
  ph.innerHTML='<img src="'+url+'">'
    +'<br><a href="'+url+'" download style="color:#4af;font-size:.9em">💾 다운로드</a>'
    +'<br><button class="b b3" onclick="document.getElementById(\'photo\').style.display=\'none\'" style="margin-top:6px;font-size:.8em">닫기</button>';
  msg('✅ 고해상도 사진 준비됨');
}

async function switchCam(n){
  msg('📷 카메라 전환 중...');
  await api('stop');
  await new Promise(r=>setTimeout(r,2000));
  await cam('3028',String(n));
  curCam=n;
  document.querySelectorAll('[id^=cam]').forEach(el=>el.classList.remove('active'));
  document.getElementById('cam'+n).classList.add('active');
  await new Promise(r=>setTimeout(r,2000));
  await go();
}

async function go(){
  ss('ld','⏳ 연결 중...');
  const r=await api('start');
  if(r==='dashcam_offline'){ss('off','❌ 대시캠 오프라인');return}
  setTimeout(play,8000);
}
function play(){
  if(h)h.destroy();
  if(!Hls.isSupported()){v.src='/hls/live.m3u8';return}
  h=new Hls({liveSyncDuration:3,liveMaxLatencyDuration:10,
    manifestLoadingRetryDelay:2000,manifestLoadingMaxRetry:30});
  h.loadSource('/hls/live.m3u8');h.attachMedia(v);
  h.on(Hls.Events.MANIFEST_PARSED,()=>{ss('on','📡 스트리밍 중');v.play()});
  h.on(Hls.Events.ERROR,(e,d)=>{if(d.type===Hls.ErrorTypes.NETWORK_ERROR)ss('ld','⏳ 버퍼링...')});
}
async function stop(){if(h){h.destroy();h=null}v.src='';await api('stop');ss('off','대기 중')}

async function getInfo(){
  const cmds=[['3012','펌웨어'],['3017','여유공간'],['3024','SD상태']];
  let info='';
  for(const [cmd,label] of cmds){
    try{
      const r=await fetch('/dashcam/?custom=1&cmd='+cmd);
      const t=await r.text();
      const val=t.match(/<Value>(.*?)<\/Value>/);
      const str=t.match(/<String>(.*?)<\/String>/);
      info+=label+': '+(str?str[1]:val?val[1]:'N/A')+' | ';
    }catch{}
  }
  msg(info||'정보 없음');
}

window.addEventListener('beforeunload',()=>navigator.sendBeacon('/api/stop'));
document.getElementById('cam0').classList.add('active');
</script>
</body>
</html>
HTML_EOF

    nginx -t && systemctl restart nginx
    systemctl enable nginx fcgiwrap

    # ======================================================
    # 모니터링 서비스들
    # ======================================================

    # --- 대시캠 Wi-Fi 자동 재연결 ---
    cat > /usr/local/bin/wifi-monitor.sh << WMON_EOF
#!/bin/bash
while true; do
    if ! ping -c 1 -W 3 "${DASHCAM_IP}" &>/dev/null; then
        nmcli con up dashcam 2>/dev/null
    fi
    sleep 30
done
WMON_EOF
    chmod +x /usr/local/bin/wifi-monitor.sh

    cat > /etc/systemd/system/wifi-monitor.service << 'SVC_EOF'
[Unit]
Description=Dashcam Wi-Fi Monitor
After=NetworkManager.service
[Service]
Type=simple
ExecStart=/usr/local/bin/wifi-monitor.sh
Restart=always
[Install]
WantedBy=multi-user.target
SVC_EOF

    # --- LTE 자동 재연결 ---
    cat > /usr/local/bin/lte-monitor.sh << 'LTE_EOF'
#!/bin/bash
while true; do
    if ! ping -c 1 -W 5 8.8.8.8 &>/dev/null; then
        nmcli con up lte 2>/dev/null
    fi
    sleep 60
done
LTE_EOF
    chmod +x /usr/local/bin/lte-monitor.sh

    cat > /etc/systemd/system/lte-monitor.service << 'SVC_EOF'
[Unit]
Description=LTE Monitor
After=ModemManager.service NetworkManager.service
[Service]
Type=simple
ExecStart=/usr/local/bin/lte-monitor.sh
Restart=always
[Install]
WantedBy=multi-user.target
SVC_EOF

    # --- 서비스 활성화 ---
    systemctl daemon-reload
    systemctl enable relay-watchdog wifi-monitor lte-monitor
    systemctl start relay-watchdog wifi-monitor lte-monitor

    # ======================================================
    # 완료
    # ======================================================
    rm -f "$PHASE_FILE"
    TS_IP=$(tailscale ip -4 2>/dev/null || echo "확인안됨")

    echo ""
    echo "============================================"
    echo "  ✅ 셋업 완료!"
    echo "============================================"
    echo ""
    echo "  Tailscale IP: $TS_IP"
    echo ""
    echo "  폰에서:"
    echo "    Tailscale 앱 + 같은 계정 로그인"
    echo "    http://${TS_IP}:8080/"
    echo ""
    echo "  SSH: ssh $(logname)@${TS_IP}"
    echo ""
    echo "  동작:"
    echo "    시동ON → 대시캠Wi-Fi → Pi자동연결"
    echo "    → LTE → Tailscale → 폰에서 접속"
    echo "    → ▶ 스트리밍 → 2분 안보면 자동중지"
    echo ""
    echo "  확인:"
    echo "    nmcli con show --active"
    echo "    tailscale status"
    echo "    mmcli -m 0"
    echo "============================================"
}

# ============================================================
# 메인
# ============================================================
if [ "$EUID" -ne 0 ]; then
    echo "sudo로 실행: sudo ./setup.sh"
    exit 1
fi

if [ ! -f "$PHASE_FILE" ]; then
    phase1
else
    phase2
fi

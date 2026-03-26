<script setup>
import PlatformTile from './components/PlatformTile.vue'
// 运行时状态：当前生成文案、选中平台、商户配置、提示状态等
let generatedText = "";
let isGenerating = false;
let selectedPlatform = "xhs";
let pendingPlatform = "";
let generatingPlatform = "";
let toastTimer = null;
let pendingAutoCopyText = "";
let merchantConfig = null;
let currentMerchantSlug = "ymdq";
// 各平台当前已生成/缓存的文案内容
const platformContentMap = {
  xhs: "",
  douyin: "",
  moments: "",
  ctrip: "",
  qunar: "",
  meituan: ""
};
// AI 开关：测试阶段可关闭真实生成，避免消耗 token
const ENABLE_AI = true;
// AI 关闭时使用的固定测试文案
const MOCK_GENERATED_TEXT = "这次入住体验很好，位置方便，房间干净舒服，整体氛围也很放松，服务贴心，来这边玩的话会愿意推荐给朋友。";

const API_BASE_URL = "https://tapwrite-api.sitarliuuu.workers.dev";

// 统一复制文本到剪贴板，优先使用 Clipboard API，失败时降级到 textarea 方案
async function writeClipboardText(text) {
  if (!text || !text.trim()) return false;

  try {
    if (navigator.clipboard && window.isSecureContext) {
      await navigator.clipboard.writeText(text);
      return true;
    }
  } catch (e) {
  }

  try {
    const textarea = document.createElement("textarea");
    textarea.value = text;
    textarea.setAttribute("readonly", "readonly");
    textarea.style.position = "fixed";
    textarea.style.top = "50%";
    textarea.style.left = "50%";
    textarea.style.width = "1px";
    textarea.style.height = "1px";
    textarea.style.padding = "0";
    textarea.style.border = "0";
    textarea.style.opacity = "0";
    textarea.style.pointerEvents = "none";
    textarea.style.transform = "translate(-50%, -50%)";
    document.body.appendChild(textarea);
    textarea.focus({preventScroll: true});
    textarea.select();
    textarea.setSelectionRange(0, textarea.value.length);
    const ok = document.execCommand("copy");
    textarea.remove();
    return !!ok;
  } catch (err) {
    return false;
  }
}

// 判断当前是否处于微信内置浏览器
function isWeChatBrowser() {
  return /MicroMessenger/i.test(window.navigator.userAgent || "");
}

// 判断某个平台操作前是否需要先展示“请在浏览器打开”的微信提示
function shouldShowWechatGuideBeforeGenerate(platform) {
  return ["xhs", "douyin", "moments", "ctrip", "qunar", "meituan"].includes(platform);
}

// 显示微信内打开限制的引导蒙层
function showWechatGuide() {
  const mask = document.getElementById("wechatMask");
  if (!mask) return;
  mask.classList.add("show");
  mask.setAttribute("aria-hidden", "false");
  document.body.style.overflow = "hidden";
}

// 关闭微信引导蒙层
function hideWechatGuide() {
  const mask = document.getElementById("wechatMask");
  if (!mask) return;
  mask.classList.remove("show");
  mask.setAttribute("aria-hidden", "true");
  document.body.style.overflow = "";
}

// 统一处理“微信内是否允许继续执行动作”的守卫逻辑
function guardWechatOpen(action, shouldBlockInWechat = true) {
  if (shouldBlockInWechat && isWeChatBrowser()) {
    showWechatGuide();
    return false;
  }
  if (typeof action === "function") {
    action();
  }
  return true;
}

// 处理暂未开放的入口点击
function handleRestrictedEntry() {
  return guardWechatOpen(() => {
    showToast("该功能正在开发中");
  }, true);
}

// 复制当前商户配置中的 WiFi 密码
async function copyWifiPassword() {
  const password = merchantConfig?.wifi?.password || "";
  return await writeClipboardText(password);
}

// 打开 WiFi 弹窗，并在打开前自动复制密码
async function openWifiModal() {
  await copyWifiPassword();
  const modal = document.getElementById("wifiModal");
  const badge = document.getElementById("wifiCopyBadge");
  if (!modal) return;

  if (badge) {
    badge.innerText = "已复制";
  }

  modal.classList.add("show");
  modal.setAttribute("aria-hidden", "false");
  document.body.style.overflow = "hidden";
}

// 关闭 WiFi 弹窗
function hideWifiModal() {
  const modal = document.getElementById("wifiModal");
  if (!modal) return;
  modal.classList.remove("show");
  modal.setAttribute("aria-hidden", "true");
  document.body.style.overflow = "";
}

// 打开微信二维码弹窗
function openWechatQrModal() {
  const modal = document.getElementById("wechatQrModal");
  if (!modal) return;
  modal.classList.add("show");
  modal.setAttribute("aria-hidden", "false");
  document.body.style.overflow = "hidden";
}

// 打开小程序码弹窗
function openMiniProgram() {
  const modal = document.getElementById("miniProgramQrModal");
  if (!modal) return;
  modal.classList.add("show");
  modal.setAttribute("aria-hidden", "false");
  document.body.style.overflow = "hidden";
}

// 关闭小程序码弹窗
function hideMiniProgramQrModal() {
  const modal = document.getElementById("miniProgramQrModal");
  if (!modal) return;
  modal.classList.remove("show");
  modal.setAttribute("aria-hidden", "true");
  document.body.style.overflow = "";
}

// 尝试拉起微信，并在需要时先复制朋友圈文案
function openWechatMoments() {
  guardWechatOpen(async () => {
    if (generatedText.trim()) {
      await writeClipboardText(generatedText);
    }

    const appUrl = "weixin://";
    const start = Date.now();
    let hasHidden = false;
    let cleanupTimer = null;
    let iframe = null;
    let link = null;

    const clearFallback = () => {
      if (cleanupTimer) {
        clearTimeout(cleanupTimer);
        cleanupTimer = null;
      }
      document.removeEventListener("visibilitychange", onVisibilityChange);
      window.removeEventListener("pagehide", clearFallback);
      window.removeEventListener("blur", clearFallback);
      if (iframe) {
        iframe.remove();
        iframe = null;
      }
      if (link) {
        link.remove();
        link = null;
      }
    };

    const onVisibilityChange = () => {
      if (document.hidden) {
        hasHidden = true;
        clearFallback();
      }
    };

    const fallbackNotice = () => {
      if (hasHidden) return;
      if (Date.now() - start < 2500) {
        showToast("已复制朋友圈文案，请手动进入微信朋友圈粘贴发布", 2600);
      }
    };

    document.addEventListener("visibilitychange", onVisibilityChange);
    window.addEventListener("pagehide", clearFallback, {once: true});
    window.addEventListener("blur", clearFallback, {once: true});

    link = document.createElement("a");
    link.href = appUrl;
    link.style.display = "none";
    document.body.appendChild(link);
    link.click();

    setTimeout(() => {
      if (hasHidden) return;
      iframe = document.createElement("iframe");
      iframe.style.display = "none";
      iframe.src = appUrl;
      document.body.appendChild(iframe);
    }, 180);

    cleanupTimer = setTimeout(() => {
      clearFallback();
      fallbackNotice();
    }, 1400);
  });
}

// 关闭微信二维码弹窗
function hideWechatQrModal() {
  const modal = document.getElementById("wechatQrModal");
  if (!modal) return;
  modal.classList.remove("show");
  modal.setAttribute("aria-hidden", "true");
  document.body.style.overflow = "";
}

// 更新页面中的结果展示区域
function setStatus(text) {
  const resultEl = document.getElementById("result");
  if (!resultEl) return;
  resultEl.classList.add("show");
  resultEl.innerText = text;
}

// 根据是否已有结果，控制结果区和操作按钮的显示状态
function updateGenerateUI(hasResult) {
  const actions = document.getElementById("resultActions");
  const resultEl = document.getElementById("result");

  if (resultEl) {
    resultEl.classList.toggle("show", !!hasResult);
  }
  if (actions) {
    actions.classList.toggle("show", !!hasResult);
  }
}

// 设置当前选中的目标平台
function setSelectedPlatform(platform) {
  selectedPlatform = platform;
}

// 显示底部轻提示
function showToast(message, duration = 2200) {
  const toast = document.getElementById("toast");
  if (!toast) return;
  toast.innerText = message;
  toast.classList.add("show");
  toast.setAttribute("aria-hidden", "false");
  if (toastTimer) clearTimeout(toastTimer);
  toastTimer = setTimeout(() => {
    toast.classList.remove("show");
    toast.setAttribute("aria-hidden", "true");
  }, duration);
}

// 根据生成状态、缓存状态，更新各平台卡片的提示文案和样式
function updatePlatformBadges() {
  const tiles = document.querySelectorAll('.tile[data-platform]');
  tiles.forEach(tile => {
    const platform = tile.getAttribute('data-platform');
    const hint = tile.querySelector('.tile-hint');
    if (!hint) return;

    const hasContent = !!(platformContentMap[platform] || "").trim();
    const isLoading = generatingPlatform === platform;
    const shouldDisable = !!generatingPlatform && generatingPlatform !== platform;
    const isQunar = platform === "qunar";

    tile.classList.toggle('is-loading', isLoading);
    tile.classList.toggle('is-disabled', shouldDisable);
    tile.classList.toggle('is-ready', hasContent && !isLoading);

    if (isLoading) {
      hint.innerText = '正在生成...';
      return;
    }

    if (hasContent) {
      hint.innerText = isQunar ? '已生成，可复制使用' : '已生成，点击可直接前往';
      return;
    }

    if (platform === 'xhs') {
      hint.innerText = '生成分享文案';
    } else if (platform === 'douyin') {
      hint.innerText = '生成评论内容';
    } else if (platform === 'moments') {
      hint.innerText = '生成朋友圈文案';
    } else if (platform === 'meituan') {
      hint.innerText = '生成评价内容';
    } else {
      hint.innerText = '生成点评内容';
    }
  });
}

// 展示“文案已生成/已复制，是否前往平台”的确认弹窗
function showPlatformConfirm(platform, copied = false) {
  const modal = document.getElementById("platformConfirmModal");
  const titleEl = document.getElementById("platformConfirmTitle");
  const descEl = document.getElementById("platformConfirmDesc");
  const goBtn = document.getElementById("platformConfirmGoBtn");
  const label = getPlatformLabel(platform);
  const isQunar = platform === "qunar";
  const isMoments = platform === "moments";

  pendingPlatform = platform;

  if (titleEl) {
    titleEl.innerText = isQunar
        ? "文案已生成"
        : (copied ? "文案已复制" : "文案已生成");
  }

  if (descEl) {
    if (isQunar) {
      descEl.innerText = copied
          ? `已为你生成并复制${label}文案。\n现在将尝试打开该店铺评论页，若未拉起 App 会自动打开网页。`
          : `已为你生成${label}文案。\n请先点击下方“复制文案”，再尝试打开该店铺评论页。`;
    } else if (isMoments) {
      descEl.innerText = copied
          ? `已为你生成并复制${label}文案\n现在可以直接前往微信粘贴发布。`
          : `已为你生成${label}文案，但系统未完成自动复制。\n请先点击下方“复制文案”，再前往微信粘贴发布。`;
    } else {
      descEl.innerText = copied
          ? `已为你生成并复制${label}文案\n现在可以直接前往${label}粘贴发布。`
          : `已为你生成${label}文案，但系统未完成自动复制。\n请先点击下方“复制文案”，再前往${label}粘贴发布。`;
    }
  }

  if (goBtn) goBtn.innerText = isQunar ? "前往去哪儿" : (isMoments ? "前往微信" : `前往${label}`);

  if (!modal) return;
  modal.classList.add("show");
  modal.setAttribute("aria-hidden", "false");
  document.body.style.overflow = "hidden";
}

// 关闭平台确认弹窗，并处理待补做的自动复制
function hidePlatformConfirm() {
  const modal = document.getElementById("platformConfirmModal");
  if (!modal) return;
  modal.classList.remove("show");
  modal.setAttribute("aria-hidden", "true");
  document.body.style.overflow = "";

  if (pendingAutoCopyText) {
    const text = pendingAutoCopyText;
    pendingAutoCopyText = "";
    setTimeout(() => {
      writeClipboardText(text).then(copied => {
        if (copied) {
          showToast("文案已自动复制");
        }
      });
    }, 80);
  }
}

// 根据当前待前往的平台，执行实际跳转逻辑
function confirmPlatformJump() {
  const platform = pendingPlatform || selectedPlatform;

  if (platform === "douyin" && pendingAutoCopyText) {
    const text = pendingAutoCopyText;
    pendingAutoCopyText = "";
    writeClipboardText(text).then(copied => {
      if (copied) {
        showToast("文案已自动复制");
      }
    });
  }

  hidePlatformConfirm();

  if (platform === "xhs") return openXHS();
  if (platform === "douyin") return openDouyin();
  if (platform === "moments") return openWechatMoments();
  if (platform === "ctrip") return openCtrip();
  if (platform === "meituan") return openMeituan();
  if (platform === "qunar") return openQunar();
}

// 处理平台入口点击：优先复用缓存文案，否则触发生成
async function handlePlatformClick(platform) {
  if (isGenerating) return;

  if (shouldShowWechatGuideBeforeGenerate(platform) && isWeChatBrowser()) {
    showWechatGuide();
    return;
  }

  setSelectedPlatform(platform);

  const cachedText = (platformContentMap[platform] || "").trim();
  if (cachedText) {
    generatedText = cachedText;
    setStatus(generatedText);
    document.getElementById("btnCopy").classList.add("show");
    updateGenerateUI(true);
    updatePlatformBadges();

    const copied = await writeClipboardText(generatedText);

    if (copied) {
      showToast(platform === "qunar" ? `${getPlatformLabel(platform)}文案已复制` : `${getPlatformLabel(platform)}文案已复制，正在打开平台`, 1800);
      pendingPlatform = platform;
      confirmPlatformJump();
    } else {
      showPlatformConfirm(platform, false);
    }
    return;
  }

  const ok = await generate();
  if (ok) {
    let copied = await writeClipboardText(generatedText);

    if (!copied && platform === "douyin") {
      pendingAutoCopyText = generatedText;
    } else {
      pendingAutoCopyText = "";
    }

    showPlatformConfirm(platform, copied);
  }
}

// 平台标识转可读中文名称
function getPlatformLabel(platform) {
  const map = {
    xhs: "小红书",
    qunar: "去哪儿",
    douyin: "抖音",
    moments: "朋友圈",
    meituan: "美团",
    ctrip: "携程"
  };
  return map[platform] || "当前平台";
}

// 生成文案：测试时返回固定文案，正式开启后调用 AI 接口
async function generate() {
  if (!ENABLE_AI) {
    generatedText = MOCK_GENERATED_TEXT;
    platformContentMap[selectedPlatform] = MOCK_GENERATED_TEXT;
    setStatus(generatedText);
    document.getElementById("btnCopy").classList.add("show");
    updateGenerateUI(true);
    updatePlatformBadges();
    return true;
  }

  if (isGenerating) return;

  if (shouldShowWechatGuideBeforeGenerate(selectedPlatform) && isWeChatBrowser()) {
    showWechatGuide();
    return false;
  }


  isGenerating = true;
  generatingPlatform = selectedPlatform;
  updatePlatformBadges();
  document.getElementById("btnGen").disabled = true;
  document.getElementById("btnCopy").disabled = true;
  document.getElementById("btnCopy").classList.remove("show");

  const platformLabel = getPlatformLabel(selectedPlatform);
  setStatus(`正在生成中...\n正在为${platformLabel}生成适配文案\n`);

  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), 20000);

  try {
    const res = await fetch(`${API_BASE_URL}/generate`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        merchantSlug: currentMerchantSlug,
        platform: selectedPlatform
      }),
      signal: controller.signal
    });
    clearTimeout(timeoutId);

    if (!res.ok) {
      let errMsg = "生成失败，请稍后再试";

      try {
        const errData = await res.json();

        if (errData?.error === "quota exceeded") {
          showToast("今日生成次数已用完，请明天再来～");
        } else if (errData?.error === "merchant disabled") {
          showToast("当前商户已停用");
        } else if (errData?.error === "rate limit exceeded") {
          showToast("操作太频繁啦，AI也需要休息一下噢😯");
        } else if (errData?.error) {
          errMsg = errData.error;
        }
      } catch (e) {
      }

      setStatus(errMsg);

      if (!generatedText.trim()) {
        updateGenerateUI(false);
      }

      return false;
    }

    const data = await res.json();
    const content = data?.content;

    if (!content) {
      setStatus("请求成功但没有拿到 content。\n\n返回：\n" + JSON.stringify(data, null, 2));
      if (!generatedText.trim()) {
        updateGenerateUI(false);
      }
      return false;
    }

    generatedText = content;
    platformContentMap[selectedPlatform] = content;
    setStatus(generatedText);
    document.getElementById("btnCopy").classList.add("show");
    updateGenerateUI(true);
    updatePlatformBadges();

    return true;

  } catch (e) {
    if (e?.name === "AbortError") {
      setStatus("请求超时，请稍后重试。");
    } else {
      setStatus("生成失败：" + (e?.message || e));
    }
    if (!generatedText.trim()) {
      updateGenerateUI(false);
    }
    return false;
  } finally {
    isGenerating = false;
    generatingPlatform = "";
    updatePlatformBadges();
    document.getElementById("btnGen").disabled = false;
    document.getElementById("btnCopy").disabled = false;
  }
}

// 重新生成当前平台文案，并在成功后尝试复制
async function regenerateContent() {
  if (!ENABLE_AI) {
    generatedText = MOCK_GENERATED_TEXT;
    platformContentMap[selectedPlatform] = MOCK_GENERATED_TEXT;
    setStatus(generatedText);
    document.getElementById("btnCopy").classList.add("show");
    updateGenerateUI(true);
    updatePlatformBadges();
  }

  const ok = await generate();
  if (!ok) return;

  let copied = await writeClipboardText(generatedText);
  const platform = selectedPlatform;
  const isQunar = platform === "qunar";
  const isDouyin = platform === "douyin";

  if (!copied && isDouyin) {
    pendingAutoCopyText = generatedText;
    showPlatformConfirm(platform, false);
    return;
  }

  pendingAutoCopyText = "";

  if (copied) {
    if (isQunar) {
      showToast(`${getPlatformLabel(platform)}文案已重新生成并复制，正在打开店铺评论页`);
      pendingPlatform = platform;
      confirmPlatformJump();
    } else {
      showToast(`${getPlatformLabel(platform)}文案已重新生成并复制`);
    }
  } else {
    showToast("文案已重新生成，但自动复制失败", 2600);
  }
}


// 复制当前结果区中的文案内容
async function copyText() {
  if (!generatedText.trim()) {
    showToast("还没有可复制的文案，请先选择平台生成");
    return;
  }

  const copied = await writeClipboardText(generatedText);
  if (copied) {
    showToast("文案已复制");
  } else {
    showToast("复制失败，请长按下方内容手动复制", 2600);
  }
}

// 打开抖音门店页，优先拉起 App，失败时回退到网页
function openDouyin() {
  guardWechatOpen(() => {
    const douyinStoreWebUrl = "https://life-share.scsjsd.com/falcon/poi_mwa/poi_detail?poi_id=7279650606363445289&app=aweme&biz_code=trip";
    const poiId = getPlatformConfig("douyin").poi_id || "";
    if (!poiId) {
      showToast("未配置抖音门店");
      return;
    }
    const appUrls = [
      "snssdk1128://poi/detail?id=" + encodeURIComponent(poiId),
      "snssdk1128://poi/detail?poi_id=" + encodeURIComponent(poiId),
      "snssdk1128://webview?url=" + encodeURIComponent(douyinStoreWebUrl)
    ];
    const start = Date.now();
    let hasHidden = false;
    let cleanupTimer = null;
    let iframe = null;
    let link = null;

    const clearFallback = () => {
      if (cleanupTimer) {
        clearTimeout(cleanupTimer);
        cleanupTimer = null;
      }
      document.removeEventListener("visibilitychange", onVisibilityChange);
      window.removeEventListener("pagehide", clearFallback);
      window.removeEventListener("blur", clearFallback);
      if (iframe) {
        iframe.remove();
        iframe = null;
      }
      if (link) {
        link.remove();
        link = null;
      }
    };

    const onVisibilityChange = () => {
      if (document.hidden) {
        hasHidden = true;
        clearFallback();
      }
    };

    const fallbackToWeb = () => {
      if (hasHidden) return;
      if (Date.now() - start < 2500) {
        window.location.href = douyinStoreWebUrl;
      }
    };

    document.addEventListener("visibilitychange", onVisibilityChange);
    window.addEventListener("pagehide", clearFallback, {once: true});
    window.addEventListener("blur", clearFallback, {once: true});

    link = document.createElement("a");
    link.href = appUrls[0];
    link.style.display = "none";
    document.body.appendChild(link);
    link.click();

    setTimeout(() => {
      if (hasHidden) return;
      iframe = document.createElement("iframe");
      iframe.style.display = "none";
      iframe.src = appUrls[1];
      document.body.appendChild(iframe);
    }, 180);

    setTimeout(() => {
      if (hasHidden || !iframe) return;
      iframe.src = appUrls[2];
    }, 420);

    cleanupTimer = setTimeout(() => {
      clearFallback();
      fallbackToWeb();
    }, 1400);
  });
}

// 打开抖音主页，优先拉起 App，失败时回退到网页
function openDouyinProfile() {
  guardWechatOpen(() => {
    const uid = getPlatformConfig("douyin").profile_uid || "";
    if (!uid) {
      showToast("未配置抖音主页");
      return;
    }
    const webUrl = "https://www.douyin.com/user/" + uid;
    const appUrls = [
      "snssdk1128://user/profile/" + encodeURIComponent(uid),
      "snssdk2329://user/profile/" + encodeURIComponent(uid),
      "aweme://user/profile/" + encodeURIComponent(uid),
      "snssdk1128://webview?url=" + encodeURIComponent(webUrl)
    ];
    const start = Date.now();
    let hasHidden = false;
    let cleanupTimer = null;
    let iframe = null;
    let link = null;

    const clearFallback = () => {
      if (cleanupTimer) {
        clearTimeout(cleanupTimer);
        cleanupTimer = null;
      }
      document.removeEventListener("visibilitychange", onVisibilityChange);
      window.removeEventListener("pagehide", clearFallback);
      window.removeEventListener("blur", clearFallback);
      if (iframe) {
        iframe.remove();
        iframe = null;
      }
      if (link) {
        link.remove();
        link = null;
      }
    };

    const onVisibilityChange = () => {
      if (document.hidden) {
        hasHidden = true;
        clearFallback();
      }
    };

    const fallbackToWeb = () => {
      if (hasHidden) return;
      if (Date.now() - start < 2500) {
        window.location.href = webUrl;
      }
    };

    document.addEventListener("visibilitychange", onVisibilityChange);
    window.addEventListener("pagehide", clearFallback, {once: true});
    window.addEventListener("blur", clearFallback, {once: true});

    link = document.createElement("a");
    link.href = appUrls[0];
    link.style.display = "none";
    document.body.appendChild(link);
    link.click();

    setTimeout(() => {
      if (hasHidden) return;
      iframe = document.createElement("iframe");
      iframe.style.display = "none";
      iframe.src = appUrls[1];
      document.body.appendChild(iframe);
    }, 180);

    setTimeout(() => {
      if (hasHidden || !iframe) return;
      iframe.src = appUrls[2];
    }, 420);

    setTimeout(() => {
      if (hasHidden || !iframe) return;
      iframe.src = appUrls[3];
    }, 680);

    cleanupTimer = setTimeout(() => {
      clearFallback();
      fallbackToWeb();
    }, 1500);
  });
}

// 打开小红书发笔记页，并尽量提前复制内容
function openXHS() {
  guardWechatOpen(() => {
    if (generatedText.trim()) {
      writeClipboardText(generatedText);
    }

    const appUrl = "xhsdiscover://post_note?ignore_draft=true";

    const link = document.createElement("a");
    link.href = appUrl;
    link.style.display = "none";
    document.body.appendChild(link);
    link.click();

    setTimeout(() => {
      link.remove();
    }, 800);
  });
}

// 打开美团门店页面
function openMeituan() {
  guardWechatOpen(() => {
    const shopId = getPlatformConfig("meituan").poi_id || "";
    if (!shopId) {
      showToast("未配置美团门店");
      return;
    }
    const hotelH5Url = "https://i.meituan.com/awp/h5/hotel/poi/deal.html?poiId=" + shopId;
    const encodedHotelH5Url = encodeURIComponent(hotelH5Url);
    const appUrl = "imeituan://www.meituan.com/web?url=" + encodedHotelH5Url;

    const link = document.createElement("a");
    link.href = appUrl;
    link.style.display = "none";
    document.body.appendChild(link);
    link.click();

    setTimeout(() => {
      const iframe = document.createElement("iframe");
      iframe.style.display = "none";
      iframe.src = appUrl;
      document.body.appendChild(iframe);

      setTimeout(() => {
        iframe.remove();
        link.remove();
      }, 1200);
    }, 150);
  });
}

// 打开去哪儿评论页，优先拉起 App，失败时回退到网页
function openQunar() {
  guardWechatOpen(() => {
    if (generatedText.trim()) {
      writeClipboardText(generatedText);
    }

    const seq = getPlatformConfig("qunar").seq || "";
    if (!seq) {
      showToast("未配置去哪儿门店");
      return;
    }
    const webUrl = "https://review.qunar.com/hotelcn/hotelcommentlist?seq=" + encodeURIComponent(seq) + "&cid=";
    const appUrls = [
      "qunartraveliphone://hy?url=" + encodeURIComponent(webUrl),
      "qunartravelaphone://hy?url=" + encodeURIComponent(webUrl)
    ];

    const start = Date.now();
    let hasHidden = false;
    let cleanupTimer = null;
    let iframe = null;
    let link = null;

    const clearFallback = () => {
      if (cleanupTimer) {
        clearTimeout(cleanupTimer);
        cleanupTimer = null;
      }
      document.removeEventListener("visibilitychange", onVisibilityChange);
      window.removeEventListener("pagehide", clearFallback);
      window.removeEventListener("blur", clearFallback);
      if (iframe) {
        iframe.remove();
        iframe = null;
      }
      if (link) {
        link.remove();
        link = null;
      }
    };

    const onVisibilityChange = () => {
      if (document.hidden) {
        hasHidden = true;
        clearFallback();
      }
    };

    const fallbackToWeb = () => {
      if (hasHidden) return;
      if (Date.now() - start < 2500) {
        window.location.href = webUrl;
      }
    };

    document.addEventListener("visibilitychange", onVisibilityChange);
    window.addEventListener("pagehide", clearFallback, {once: true});
    window.addEventListener("blur", clearFallback, {once: true});

    link = document.createElement("a");
    link.href = appUrls[0];
    link.style.display = "none";
    document.body.appendChild(link);
    link.click();

    setTimeout(() => {
      if (hasHidden) return;
      iframe = document.createElement("iframe");
      iframe.style.display = "none";
      iframe.src = appUrls[1];
      document.body.appendChild(iframe);
    }, 180);

    cleanupTimer = setTimeout(() => {
      clearFallback();
      fallbackToWeb();
    }, 1400);
  });
}

// 打开携程酒店详情页，优先拉起 App，失败时回退到 H5
function openCtrip() {
  guardWechatOpen(() => {
    const hotelId = getPlatformConfig("ctrip").hotel_id || "";
    if (!hotelId) {
      showToast("未配置携程酒店");
      return;
    }
    const h5Url = "https://m.ctrip.com/webapp/hotel/hoteldetail/" + hotelId + ".html";
    const encodedH5ForApp = btoa(unescape(encodeURIComponent(h5Url)));
    const appUrl = "ctrip://wireless/h5?url=" + encodedH5ForApp + "&type=2";
    const start = Date.now();

    const fallbackTimer = setTimeout(() => {
      if (Date.now() - start < 1800) {
        window.location.href = h5Url;
      }
    }, 1200);

    const clearFallback = () => {
      clearTimeout(fallbackTimer);
      document.removeEventListener("visibilitychange", onVisibilityChange);
      window.removeEventListener("pagehide", clearFallback);
      window.removeEventListener("blur", clearFallback);
    };

    const onVisibilityChange = () => {
      if (document.hidden) {
        clearFallback();
      }
    };

    document.addEventListener("visibilitychange", onVisibilityChange);
    window.addEventListener("pagehide", clearFallback, {once: true});
    window.addEventListener("blur", clearFallback, {once: true});

    window.location.href = appUrl;
  });
}

// 根据商户 slug 加载对应 JSON 配置
async function loadMerchantConfig(slug) {
  try {
    const res = await fetch(`${API_BASE_URL}/config?m=${encodeURIComponent(slug)}`);

    if (!res.ok) {
      throw new Error("NOT_FOUND");
    }

    merchantConfig = await res.json();
    currentMerchantSlug = slug;
    applyMerchantConfig();

  } catch (e) {
    console.error("商户加载失败:", slug);

    document.body.innerHTML = `
            <div style="text-align:center;margin-top:60px;">
                <h2>商户不存在</h2>
                <p style="color:#888;">请检查访问链接是否正确</p>
            </div>
        `;
  }
}

// 将商户配置应用到页面：标题、文案、二维码、WiFi、功能开关等
function applyMerchantConfig() {
  if (!merchantConfig) return;

  const merchant = merchantConfig?.merchant;
  if (!merchant || merchant.enabled === false) {
    document.body.innerHTML = `
                <div style="text-align:center;margin-top:60px;">
                    <h2>商户已停用</h2>
                    <p style="color:#888;">当前页面暂不可访问</p>
                </div>
            `;
    return;
  }
  const wifi = merchantConfig.wifi || {};
  const social = merchantConfig.social || {};

  document.title = merchant.page_title || document.title;

  const panelTitle = document.querySelector(".panel-title");
  if (panelTitle) {
    panelTitle.innerText = merchant.welcome_title || "";
  }

  const panelDesc = document.querySelector(".panel-desc");
  if (panelDesc) {
    panelDesc.innerText = merchant.welcome_desc || "";
  }

  const hintBlock = document.getElementById("hintBlock");
  const hints = merchant?.ui?.hints || merchantConfig?.ui?.hints || [];
  if (hintBlock) {
    hintBlock.innerHTML = hints.map(text => `<div>${text}</div>`).join("");
  }

  const wifiRows = document.querySelectorAll("#wifiModal .wifi-info-text");
  if (wifiRows[0]) {
    wifiRows[0].innerText = wifi.ssid || "";
  }
  if (wifiRows[1]) {
    wifiRows[1].innerText = wifi.password || "";
  }

  const wechatQrImg = document.querySelector("#wechatQrModal .wechat-qr-image");
  if (wechatQrImg && social.wechat_qr) {
    wechatQrImg.src = social.wechat_qr;
  }

  const miniProgramQrImg = document.querySelector("#miniProgramQrModal .wechat-qr-image");
  if (miniProgramQrImg && social.mini_program_qr) {
    miniProgramQrImg.src = social.mini_program_qr;
  }
  const features = merchantConfig.features || {};
  const featureTiles = document.querySelectorAll("[data-feature]");

  featureTiles.forEach(tile => {
    const key = tile.getAttribute("data-feature");
    if (features[key] === false) {
      tile.style.display = "none";
    } else {
      tile.style.display = "";
    }
  });
}

// 读取某个平台在当前商户下的配置参数
function getPlatformConfig(platform) {
  return merchantConfig?.platforms?.[platform]
      || merchantConfig?.merchant?.platforms?.[platform]
      || {};
}

// 页面初始化：从 URL 读取商户参数，加载配置并刷新 UI
async function initApp() {
  console.log("当前商户：", currentMerchantSlug);
  try {
    const params = new URLSearchParams(window.location.search);
    const slug = params.get("m");
    if (!slug) {
      document.body.innerHTML = "<h2 style='text-align:center;margin-top:40px;'>商户不存在,请检查访问链接是否正确</h2>";
      return;
    }
    await loadMerchantConfig(slug);
  } catch (e) {
    console.error(e);
    showToast("商户配置加载失败", 3000);
  }

  updateGenerateUI(false);
  updatePlatformBadges();
}

initApp();
updatePlatformBadges();
</script>

<template>
  <div class="page">

    <section class="panel">
      <div class="panel-kicker">入住快捷助手vue版</div>
      <h3 class="panel-title">欢迎入住游牧地球民宿</h3>
      <p class="panel-desc">
        很高兴与你相遇，愿你在这里住得舒服、玩得开心。下方为你准备了入住期间常用的快捷服务，点击即可使用。</p>

      <div class="hint" id="hintBlock"></div>

      <div id="result"></div>

      <div id="resultActions" class="toolbar result-actions">
        <button id="btnGen" @click="regenerateContent()">重新生成</button>
        <button id="btnCopy" class="copy-icon-btn"  @click="copyText()" aria-label="复制文案" title="复制文案">
          复制文案
        </button>
      </div>
    </section>

    <section class="panel">
      <div class="grid">
        <PlatformTile
            platform="xhs"
            title="小红书"
            icon="/images/xhs.png"
            iconClass="icon-xhs"
            :hintText="'生成分享文案'"
            :isLoading="generatingPlatform === 'xhs'"
            :isDisabled="generatingPlatform && generatingPlatform !== 'xhs'"
            :hasContent="platformContentMap['xhs']"
            @tile-click="handlePlatformClick"
        />
        <PlatformTile
            platform="douyin"
            title="抖音写评论"
            icon="/images/douyin.png"
            iconClass="icon-dy"
            :hintText="'生成评论内容'"
            :isLoading="generatingPlatform === 'douyin'"
            :isDisabled="generatingPlatform && generatingPlatform !== 'douyin'"
            :hasContent="platformContentMap['douyin']"
            @tile-click="handlePlatformClick"
        />
        <PlatformTile
            platform="moments"
            title="发朋友圈"
            icon="/images/moments.png"
            iconClass="icon-wx"
            :hintText="'生成朋友圈文案'"
            :isLoading="generatingPlatform === 'moments'"
            :isDisabled="generatingPlatform && generatingPlatform !== 'moments'"
            :hasContent="platformContentMap['moments']"
            @tile-click="handlePlatformClick"
        />
        <PlatformTile
            platform="ctrip"
            title="携程"
            icon="/images/ctrip.png"
            iconClass="icon-ctrip"
            :hintText="'生成点评内容'"
            :isLoading="generatingPlatform === 'ctrip'"
            :isDisabled="generatingPlatform && generatingPlatform !== 'ctrip'"
            :hasContent="platformContentMap['ctrip']"
            @tile-click="handlePlatformClick"
        />
        <PlatformTile
            platform="qunar"
            title="去哪儿"
            icon="/images/qunar.png"
            iconClass="icon-qunar"
            :hintText="'生成点评内容'"
            :isLoading="generatingPlatform === 'qunar'"
            :isDisabled="generatingPlatform && generatingPlatform !== 'qunar'"
            :hasContent="platformContentMap['qunar']"
            @tile-click="handlePlatformClick"
        />
        <PlatformTile
            platform="meituan"
            title="美团"
            icon="/images/meituan.png"
            iconClass="icon-mt"
            :hintText="'生成评价内容'"
            :isLoading="generatingPlatform === 'meituan'"
            :isDisabled="generatingPlatform && generatingPlatform !== 'meituan'"
            :hasContent="platformContentMap['meituan']"
            @tile-click="handlePlatformClick"
        />
        <div class="tile" data-feature="video_channel" @click="handleRestrictedEntry()">
          <div class="tile-icon icon-channel"><img src="/images/wx_channel.png" alt="视频号"/></div>
          <h3>关注视频号</h3>
        </div>
        <div class="tile" data-feature="douyin_profile" @click="openDouyinProfile()">
          <div class="tile-icon icon-dy"><img src="/images/douyin.png" alt="关注抖音"/></div>
          <h3>关注抖音</h3>
        </div>
        <div class="tile mini-program-tile" data-feature="mini_program" @click="openMiniProgram()">
          <div class="tile-icon icon-wx"><img src="/images/miniapp.png" alt="一屿藏饰手作"/></div>
          <h3>一屿藏饰手作</h3>
        </div>
        <div class="tile" data-feature="wechat_qr" @click="openWechatQrModal()">
          <div class="tile-icon icon-wx"><img src="/images/wechat.png" alt="加微信"/></div>
          <h3>加微信</h3>
        </div>
        <div class="tile" data-feature="wifi" @click="openWifiModal()">
          <div class="tile-icon icon-wifi"><img src="../images/wifi.png" alt="连WIFI"/></div>
          <h3>连WIFI</h3>
        </div>
        <div class="tile" data-feature="more" @click="showToast('噔噔噔噔~敬请期待🤓')">
          <div class="tile-icon icon-dp"><img src="/images/more.png" alt="更多"/></div>
          <h3>更多</h3>
        </div>
      </div>
    </section>
  </div>
  <div id="platformConfirmModal" class="platform-confirm-overlay" aria-hidden="true">
    <div class="platform-confirm-card">
      <h2 class="platform-confirm-title" id="platformConfirmTitle">文案已复制</h2>
      <p class="platform-confirm-desc" id="platformConfirmDesc">你现在可以前往对应平台粘贴发布。</p>
      <div class="platform-confirm-actions">
        <button type="button" class="platform-confirm-cancel" @click="hidePlatformConfirm()">取消</button>
        <button type="button" id="platformConfirmGoBtn" @click="confirmPlatformJump()">前往平台</button>
      </div>
    </div>
  </div>

  <div id="wechatMask" class="wx-mask" aria-hidden="true">
    <div class="wx-mask-inner">
      <button class="wx-mask-close" type="button" @click="hideWechatGuide()">×</button>
      <button class="wx-mask-more" type="button" aria-label="更多">···</button>

      <svg class="wx-arrow" viewBox="0 0 120 220" fill="none" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
        <path d="M20 190C20 132 42 112 76 108C108 104 111 72 108 24" stroke="#b9bbc3" stroke-width="3"
              stroke-linecap="round" stroke-dasharray="9 10"/>
        <path d="M100 34L108 18L116 34" stroke="#b9bbc3" stroke-width="3" stroke-linecap="round"
              stroke-linejoin="round"/>
      </svg>

      <h2 class="wx-mask-title">还差1步得好礼</h2>
      <p class="wx-mask-desc">
        由于微信限制，请点击右上角 <span
          style="display:inline-block;background:#e5e5e7;color:#666;border-radius:10px;padding:0 10px;line-height:1.5;">···</span><br>
        选择 <span class="highlight">“在浏览器打开”</span>
      </p>

      <div class="wx-phone" aria-hidden="true">
        <div class="wx-phone-notch"></div>
        <div class="wx-phone-sheet"></div>
        <div class="wx-phone-grid">
          <span></span><span></span><span></span><span></span>
          <span></span><span></span><span></span><span></span>
        </div>
        <div class="wx-phone-focus">
          <div class="wx-phone-focus-inner">
            <div class="wx-browser-icon">🌐</div>
            在浏览器打开
          </div>
        </div>
      </div>

      <button class="wx-mask-bottom-close" type="button" @click="hideWechatGuide()">×</button>
    </div>
  </div>
  <div id="wifiModal" class="wifi-modal" aria-hidden="true">
    <div class="wifi-modal-card">
      <h2 class="wifi-modal-title">已复制密码，去系统页连接吧</h2>

      <div class="wifi-info-row">
        <div class="wifi-info-icon">📶</div>
        <div class="wifi-info-text">游牧地球民宿</div>
      </div>

      <div class="wifi-info-row wifi-password-row">
        <div class="wifi-info-icon">🔒</div>
        <div class="wifi-info-text">ymdq888888</div>
        <span class="wifi-copy-badge" id="wifiCopyBadge">已复制</span>
      </div>

      <button class="wifi-modal-close" type="button" @click="hideWifiModal()">×</button>
    </div>
  </div>
  <div id="wechatQrModal" class="wifi-modal" aria-hidden="true">
    <div class="wifi-modal-card wechat-qr-card">
      <h2 class="wifi-modal-title">长按识别二维码添加微信</h2>
      <p class="wechat-qr-desc">若无法直接识别，可截图后打开微信扫一扫，从相册中选择二维码。</p>

      <div class="wechat-qr-box">
        <img class="wechat-qr-image" src="/images/ymdq/wechat_qr.png" alt="微信二维码"/>
      </div>

      <button class="wifi-modal-close" type="button" @click="hideWechatQrModal()">×</button>
    </div>
  </div>
  <div id="miniProgramQrModal" class="wifi-modal" aria-hidden="true">
    <div class="wifi-modal-card wechat-qr-card">
      <h2 class="wifi-modal-title">长按识别二维码打开小程序</h2>
      <p class="wechat-qr-desc">若无法直接识别，可截图后打开微信扫一扫，从相册中选择小程序码。</p>

      <div class="wechat-qr-box">
        <img class="wechat-qr-image" src="/images/ymdq/miniProgram_qr.png" alt="小程序二维码"/>
      </div>

      <button class="wifi-modal-close" type="button" @click="hideMiniProgramQrModal()">×</button>
    </div>
  </div>
  <div id="toast" class="toast" aria-hidden="true"></div>
</template>

<style>
* {
  box-sizing: border-box;
}

body {
  margin: 0;
  font-family: -apple-system, BlinkMacSystemFont, "PingFang SC", "Helvetica Neue", Arial;
  background: #eef3ff url('/images/background.png') no-repeat center top;
  background-size: cover;
  color: #111;
  padding: 20px 16px 32px;
  text-align: center;
}

.page {
  max-width: 520px;
  margin: 0 auto;
}

.hero {
  text-align: center;
  background: url('/public/images/banner.png') no-repeat center;
  background-size: cover;
  border-radius: 24px;
  padding: 32px 20px 24px;
  margin-bottom: 18px;
  position: relative;
}

.hero::before {
  content: "";
  position: absolute;
  inset: 0;
  background: rgba(255, 255, 255, 0.72);
  border-radius: 24px;
}

.hero > * {
  position: relative;
  z-index: 1;
}

.hero h1 {
  margin: 0;
  font-size: 34px;
  line-height: 1.2;
  color: #1d2755;
  font-weight: 700;
  letter-spacing: 1px;
}

.hero p {
  margin: 12px 0 0;
  font-size: 16px;
  color: #5a648a;
}

.panel-kicker {
  margin: 0 0 8px;
  font-size: 12px;
  line-height: 1;
  font-weight: 600;
  color: #5a648a;
  letter-spacing: 1px;
}

.panel {
  background: #fff;
  border-radius: 24px;
  padding: 18px;
  box-shadow: 0 10px 28px rgba(29, 39, 85, .08);
  margin-bottom: 18px;
  text-align: left;
}

.panel-title {
  margin: 0 0 6px;
  font-size: 18px;
  font-weight: 700;
  color: #111;
}

.panel-desc {
  margin: 0;
  font-size: 13px;
  color: #888;
  line-height: 1.5;
}

.grid {
  display: grid;
  grid-template-columns:repeat(3, minmax(0, 1fr));
  gap: 16px;
}

.tile {
  background: #fff;
  border-radius: 24px;
  padding: 18px 16px 16px;
  box-shadow: 0 10px 28px rgba(29, 39, 85, .08);
  text-align: left;
  cursor: pointer;
  position: relative;
  border: 1px solid transparent;
  transition: transform .18s ease, opacity .18s ease, box-shadow .18s ease, border-color .18s ease, background .18s ease;
}

.tile.is-loading {
  transform: translateY(-1px);
  box-shadow: 0 14px 32px rgba(29, 39, 85, .12);
  border-color: #e7d9aa;
  background: #fffdf7;
}

.tile.is-ready {
  border-color: #d8e3f8;
  background: #f8fbff;
}

.tile.is-disabled {
  opacity: .45;
  pointer-events: none;
}

.tile-icon {
  width: 56px;
  height: 56px;
  border-radius: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 14px;
}

.tile-icon img {
  width: 28px;
  height: 28px;
  object-fit: contain;
}

.icon-xhs {
  background: #ffe8ed;
}

.icon-dp {
  background: #fff0e6;
}

.icon-ctrip {
  background: #e8f1ff;
}

.icon-qunar {
  background: #e8f7f7;
}

.icon-dy {
  background: #eef1ff;
}

.icon-mt {
  background: #fff7cf;
}

.icon-wifi {
  background: #e8f3ff;
}

.icon-wx {
  background: #e9f9e8;
}

.icon-channel {
  background: #eef6ef;
}

.tile h3 {
  margin: 0;
  font-size: 16px;
  line-height: 1.35;
  color: #111;
  text-align: center;
}

/* 避免平台名称过长换行 */
.tile[data-platform="douyin"] h3,
.tile.mini-program-tile h3 {
  font-size: 14px;
  white-space: nowrap;
}

.tile p {
  margin: 8px 0 0;
  font-size: 12px;
  line-height: 1.5;
  color: #7f879d;
  min-height: 18px;
  text-align: center;
}

button {
  width: 100%;
  padding: 12px 12px;
  margin-top: 12px;
  border: none;
  border-radius: 14px;
  font-size: 14px;
  font-weight: 600;
  background: #ff2442;
  color: #fff;
}

button.secondary {
  background: #111;
}

.copy-icon-btn {
  display: none;
  width: 100%;
  height: auto;
  padding: 12px 12px;
  margin-top: 0;
  border: none;
  border-radius: 14px;
  background: #111;
  color: #fff;
  font-size: 14px;
  font-weight: 600;
  line-height: 1.2;
  align-items: center;
  justify-content: center;
  justify-self: stretch;
}

.copy-icon-btn.show {
  display: inline-flex;
}

.copy-icon-img {
  display: none;
}

button:disabled {
  opacity: .6;
}

.toolbar {
  display: grid;
  grid-template-columns:1fr auto;
  gap: 10px;
  margin-top: 14px;
}

.toolbar button {
  margin-top: 0;
}

.result-actions {
  display: none;
}

.result-actions.show {
  display: grid;
}

input {
  width: 100%;
  box-sizing: border-box;
  padding: 12px 14px;
  border-radius: 14px;
  border: 1px solid #d9def0;
  font-size: 14px;
  margin-top: 12px;
  outline: none;
  background: #fff;
}

#result {
  display: none;
  white-space: pre-wrap;
  text-align: left;
  margin-top: 14px;
  padding: 14px;
  background: #f7f8fc;
  border-radius: 16px;
  min-height: 120px;
  font-size: 14px;
  line-height: 1.6;
  color: #111;
  border: 1px solid #edf0f7;
}

#result.show {
  display: block;
}

.hint {
  font-size: 12px;
  color: #666;
  text-align: left;
  margin-top: 12px;
  line-height: 1.6;
}

.section-title {
  margin: 0 0 12px;
  font-size: 16px;
  font-weight: 700;
  color: #32406d;
}

.wx-mask {
  position: fixed;
  inset: 0;
  background: linear-gradient(180deg, #f4f4f6 0%, #eef1f7 100%);
  display: none;
  align-items: flex-start;
  justify-content: center;
  padding: 40px 24px 28px;
  z-index: 9999;
  overflow: hidden;
}

.wx-mask.show {
  display: flex;
}

.wx-mask-inner {
  width: 100%;
  max-width: 420px;
  text-align: center;
  position: relative;
  color: #2f2f2f;
  transform: translateY(-16%);
  display: flex;
  flex-direction: column;
  align-items: center;
}

.wx-mask-close {
  display: none;
}

.wx-mask-more {
  position: absolute;
  right: 0;
  top: 2px;
  width: 48px;
  height: 36px;
  border: none;
  border-radius: 12px;
  background: #e5e5e7;
  color: #666;
  font-size: 24px;
  line-height: 36px;
  padding: 0;
  margin: 0;
  cursor: default;
}

.wx-mask-title {
  margin: 88px 0 18px;
  font-size: 30px;
  line-height: 1.2;
  font-weight: 700;
  color: #2f2f2f;
}

.wx-mask-desc {
  margin: 0;
  font-size: 17px;
  line-height: 1.9;
  color: #4a4a4a;
}

.wx-mask-desc .highlight {
  color: #1677ff;
  font-weight: 600;
}

.wx-arrow {
  position: absolute;
  right: 20px;
  top: 84px;
  width: 96px;
  height: 180px;
  pointer-events: none;
}

.wx-phone {
  width: 220px;
  height: 400px;
  margin: 28px auto 0;
  border: 10px solid rgba(255, 255, 255, .9);
  border-radius: 38px;
  position: relative;
  background: rgba(255, 255, 255, .22);
  box-shadow: inset 0 0 0 1px rgba(255, 255, 255, .35);
  flex: 0 0 auto;
}

.wx-phone-notch {
  position: absolute;
  left: 50%;
  top: 0;
  transform: translateX(-50%);
  width: 92px;
  height: 18px;
  background: rgba(255, 255, 255, .92);
  border-radius: 0 0 14px 14px;
}

.wx-phone-sheet {
  position: absolute;
  left: 12px;
  right: 12px;
  bottom: 16px;
  height: 150px;
  border-radius: 24px;
  background: rgba(255, 255, 255, .28);
}

.wx-phone-grid {
  position: absolute;
  left: 26px;
  right: 26px;
  bottom: 42px;
  display: grid;
  grid-template-columns:repeat(4, 1fr);
  gap: 14px;
}

.wx-phone-grid span {
  display: block;
  height: 38px;
  border-radius: 8px;
  background: rgba(255, 255, 255, .48);
}

.wx-phone-focus {
  position: absolute;
  left: 50%;
  bottom: 96px;
  transform: translateX(-50%);
  width: 144px;
  height: 144px;
  border-radius: 50%;
  border: 8px solid rgba(255, 255, 255, .75);
  background: rgba(255, 255, 255, .18);
  display: flex;
  align-items: center;
  justify-content: center;
}

.wx-phone-focus-inner {
  width: 88px;
  text-align: center;
  color: #4a4a4a;
  font-size: 14px;
  line-height: 1.35;
  font-weight: 600;
}

.wx-browser-icon {
  width: 44px;
  height: 44px;
  margin: 0 auto 8px;
  border-radius: 14px;
  background: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 4px 12px rgba(0, 0, 0, .06);
  font-size: 24px;
}

.wx-mask-bottom-close {
  width: 54px;
  height: 54px;
  margin: 20px auto 0;
  border-radius: 50%;
  border: 2px solid #8f8f94;
  background: transparent;
  color: #5a5a5f;
  font-size: 28px;
  line-height: 50px;
  padding: 0;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex: 0 0 auto;
}

@media (max-height: 760px) {
  .wx-mask {
    padding: 28px 20px 22px;
  }

  .wx-mask-inner {
    transform: translateY(-12%);
  }

  .wx-mask-title {
    margin-top: 72px;
  }

  .wx-phone {
    height: 360px;
  }

  .wx-mask-bottom-close {
    margin-top: 14px;
  }
}

@media (max-width: 420px) {
  .platform-confirm-card {
    border-radius: 20px;
    padding: 22px 18px 18px;
  }

  .platform-confirm-title {
    font-size: 18px;
  }

  .platform-confirm-desc {
    font-size: 13px;
  }

  .hero h1 {
    font-size: 28px;
  }

  .grid {
    grid-template-columns:repeat(3, minmax(0, 1fr));
    gap: 12px;
  }

  .tile {
    padding: 16px 14px;
  }

  .wifi-modal-card {
    border-radius: 28px;
    padding: 34px 20px 28px;
  }

  .wifi-modal-title {
    font-size: 18px;
    margin-bottom: 22px;
  }

  .wifi-info-row {
    min-height: 60px;
    padding: 0 14px;
    gap: 8px;
  }

  .wifi-info-icon {
    width: 24px;
    flex-basis: 24px;
    font-size: 16px;
  }

  .wifi-info-text {
    font-size: 12px;
  }

  .copy-icon-btn {
    width: 100%;
    height: auto;
    font-size: 14px;
    border-radius: 14px;
  }

  .copy-icon-img {
    display: none;
  }

  .wifi-copy-badge {
    min-width: 78px;
    height: 44px;
    font-size: 14px;
    padding: 0 12px;
  }

  .wifi-modal-close {
    width: 58px;
    height: 58px;
    font-size: 30px;
  }

  .wechat-qr-desc {
    font-size: 13px;
    margin: -6px 0 18px;
  }

  .wechat-qr-box {
    border-radius: 20px;
    padding: 14px;
  }

  .wechat-qr-image {
    max-width: 280px;
    border-radius: 14px;
  }
}

.platform-confirm-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, .42);
  display: none;
  align-items: center;
  justify-content: center;
  padding: 24px;
  z-index: 10001;
}

.platform-confirm-overlay.show {
  display: flex;
}

.platform-confirm-card {
  width: 100%;
  max-width: 420px;
  background: #fff;
  border-radius: 24px;
  padding: 26px 22px 22px;
  text-align: center;
  box-shadow: 0 18px 50px rgba(0, 0, 0, .18);
}

.platform-confirm-title {
  margin: 0 0 10px;
  font-size: 20px;
  line-height: 1.4;
  color: #111;
  font-weight: 700;
}

.platform-confirm-desc {
  margin: 0;
  font-size: 14px;
  line-height: 1.75;
  color: #666;
  white-space: pre-line;
}

.platform-confirm-actions {
  display: grid;
  grid-template-columns:1fr 1fr;
  gap: 10px;
  margin-top: 18px;
}

.platform-confirm-actions button {
  margin-top: 0;
}

.platform-confirm-cancel {
  background: #f3f4f7;
  color: #333;
}

.wifi-modal {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, .42);
  display: none;
  align-items: center;
  justify-content: center;
  padding: 24px;
  z-index: 10000;
}

.wifi-modal.show {
  display: flex;
}

.wifi-modal-card {
  width: 100%;
  max-width: 640px;
  background: #f4f4f5;
  border-radius: 32px;
  padding: 44px 28px 34px;
  position: relative;
  box-shadow: 0 18px 50px rgba(0, 0, 0, .18);
}

.wifi-modal-title {
  margin: 0 0 28px;
  text-align: center;
  font-size: 22px;
  line-height: 1.4;
  color: #222;
  font-weight: 600;
}

.wifi-info-row {
  min-height: 76px;
  border-radius: 20px;
  background: #fff;
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 0 18px;
  margin-bottom: 14px;
}

.wifi-password-row {
  margin-bottom: 0;
}

.wifi-info-icon {
  width: 32px;
  text-align: center;
  font-size: 22px;
  line-height: 1;
  flex: 0 0 32px;
}

.wifi-info-text {
  font-size: 16px;
  line-height: 1.4;
  color: #222;
  word-break: break-all;
  flex: 1;
  text-align: left;
}

.wifi-copy-badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 96px;
  height: 52px;
  padding: 0 16px;
  border-radius: 12px;
  background: #dff4df;
  color: #44a35f;
  font-size: 16px;
  font-weight: 600;
  flex: 0 0 auto;
}

.wifi-modal-close {
  width: 64px;
  height: 64px;
  margin: 26px auto 0;
  border-radius: 50%;
  border: 3px solid rgba(255, 255, 255, .92);
  background: rgba(0, 0, 0, .18);
  color: #fff;
  font-size: 34px;
  line-height: 1;
  padding: 0;
  display: flex;
  align-items: center;
  justify-content: center;
}

.wechat-qr-card {
  max-width: 460px;
  text-align: center;
}

.wechat-qr-desc {
  margin: -8px 0 22px;
  font-size: 14px;
  line-height: 1.7;
  color: #666;
  text-align: center;
}

.wechat-qr-box {
  background: #fff;
  border-radius: 24px;
  padding: 18px;
  box-shadow: inset 0 0 0 1px #f0f0f0;
}

.wechat-qr-image {
  display: block;
  width: 100%;
  max-width: 320px;
  margin: 0 auto;
  border-radius: 16px;
  object-fit: contain;
  background: #fff;
}

.toast {
  position: fixed;
  left: 50%;
  bottom: 28px;
  transform: translateX(-50%) translateY(10px);
  max-width: calc(100% - 32px);
  padding: 10px 14px;
  border-radius: 999px;
  background: rgba(17, 17, 17, .92);
  color: #fff;
  font-size: 13px;
  line-height: 1.4;
  box-shadow: 0 10px 24px rgba(0, 0, 0, .18);
  opacity: 0;
  pointer-events: none;
  transition: opacity .18s ease, transform .18s ease;
  z-index: 10003;
}

.toast.show {
  opacity: 1;
  transform: translateX(-50%) translateY(0);
}
</style>
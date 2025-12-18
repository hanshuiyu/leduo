// ==UserScript==
// @name         页面资源嗅探器 (定制版 v2.7)
// @namespace    http://tampermonkey.net/
// @version      2.7
// @description  专用于 pmplatform。增加了对 <span filesrc="..."> 这种特殊文件链接的嗅探支持。支持拖拽、清空、批量下载。
// @author       You
// @match        https://pmplatform.chinese.cn/*
// @grant        GM_download
// @grant        GM_setClipboard
// @grant        GM_addStyle
// @run-at       document-end
// ==/UserScript==

(function() {
    'use strict';

    // === 配置项 ===
    const TARGET_EXTENSIONS = ['.pdf', '.jpg', '.jpeg', '.png', '.gif', '.webp', '.svg', '.bmp', '.doc', '.docx', '.xls', '.xlsx', '.zip', '.rar'];

    // === 全局变量 ===
    let foundResources = [];
    let seenUrls = new Set();
    let observer = null;
    let isObserving = false;

    // === 样式注入 ===
    GM_addStyle(`
        #sniff-floater {
            position: fixed; bottom: 50px; right: 20px; z-index: 2147483647 !important;
            width: 50px; height: 50px; background: #007bff; border-radius: 50%;
            color: white; text-align: center; line-height: 50px; cursor: pointer;
            box-shadow: 0 4px 10px rgba(0,0,0,0.5); font-size: 24px; user-select: none;
            transition: transform 0.2s;
        }
        #sniff-floater:hover { transform: scale(1.1); }
        #sniff-floater.recording { background: #dc3545; animation: pulse 1.5s infinite; }
        @keyframes pulse {
            0% { box-shadow: 0 0 0 0 rgba(220, 53, 69, 0.7); }
            70% { box-shadow: 0 0 0 15px rgba(220, 53, 69, 0); }
            100% { box-shadow: 0 0 0 0 rgba(220, 53, 69, 0); }
        }
        #sniff-panel {
            display: none; position: fixed; top: 10%; left: 50%;
            transform: translateX(-50%);
            width: 700px; max-height: 80vh; background: white; z-index: 2147483647 !important;
            border-radius: 8px; box-shadow: 0 0 50px rgba(0,0,0,0.5);
            flex-direction: column; overflow: hidden; font-family: sans-serif; border: 1px solid #ccc;
        }
        #sniff-header {
            padding: 15px; background: #f8f9fa; border-bottom: 1px solid #dee2e6;
            display: flex; justify-content: space-between; align-items: center;
            cursor: move; user-select: none;
        }
        #sniff-controls {
            padding: 10px 15px; border-bottom: 1px solid #eee; background: #fff;
            display: flex; align-items: center; gap: 10px; flex-wrap: wrap;
        }
        .sniff-btn {
            padding: 6px 12px; border: none; border-radius: 4px; cursor: pointer;
            font-size: 13px; color: white; transition: opacity 0.2s; white-space: nowrap;
        }
        .btn-scan { background: #28a745; }
        .btn-monitor { background: #6c757d; }
        .btn-monitor.active { background: #dc3545; }
        .btn-clear { background: #dc3545; }
        .btn-dl-sel { background: #17a2b8; display: none; }

        .batch-ops { display: flex; align-items: center; margin-left: auto; gap: 8px; font-size: 13px; color: #555; }
        .batch-ops input[type="checkbox"] { transform: scale(1.2); cursor: pointer; }

        #sniff-content {
            flex: 1; overflow-y: auto; padding: 10px; background: #fdfdfd; min-height: 200px;
        }
        .sniff-item {
            display: flex; align-items: center; padding: 8px; border-bottom: 1px solid #eee; font-size: 13px; color: #333;
            animation: fadeIn 0.3s;
        }
        .sniff-item:hover { background: #f1f1f1; }
        .item-check { margin-right: 12px; transform: scale(1.3); cursor: pointer; }
        .item-preview { width: 40px; height: 40px; object-fit: contain; margin-right: 10px; background: #eee; border: 1px solid #ddd; }
        .item-info { flex: 1; overflow: hidden; }
        .item-name {
            white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
            color: #333; font-weight: bold; display: block; margin-bottom: 2px;
            max-width: 450px;
        }
        .item-url {
            white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
            color: #999; font-size: 11px; display: block; max-width: 450px; text-decoration: none;
        }
        .mini-btn { padding: 2px 8px; border: 1px solid #ccc; background: white; cursor: pointer; border-radius: 3px; font-size: 12px; margin-left: 5px;}
        .mini-btn:hover { background: #eee; }
    `);

    // === UI 创建 ===
    function createUI() {
        if (document.getElementById('sniff-floater')) return;

        const floater = document.createElement('div');
        floater.id = 'sniff-floater';
        floater.innerHTML = '🔍';
        document.body.appendChild(floater);

        const panel = document.createElement('div');
        panel.id = 'sniff-panel';
        panel.innerHTML = `
            <div id="sniff-header" title="按住此处拖拽窗口">
                <h3 style="margin:0; font-size:16px;">资源嗅探器 v2.7</h3>
                <span id="sniff-close" style="cursor:pointer; font-size:20px;">×</span>
            </div>
            <div id="sniff-controls">
                <button id="btn-scan" class="sniff-btn btn-scan">⚡ 扫描</button>
                <button id="btn-monitor" class="sniff-btn btn-monitor">🔴 监听</button>
                <button id="btn-clear" class="sniff-btn btn-clear">🗑️ 清空</button>
                <div class="batch-ops">
                    <label style="cursor:pointer; user-select:none;">
                        <input type="checkbox" id="check-all"> 全选
                    </label>
                    <button id="btn-dl-sel" class="sniff-btn btn-dl-sel">📥 下载选中</button>
                </div>
            </div>
            <div id="sniff-content">
                <div id="empty-tip" style="text-align:center; color:#999; margin-top:50px;">
                    <p>列表空空如也</p>
                </div>
            </div>
        `;
        document.body.appendChild(panel);

        // 事件绑定
        floater.onclick = (e) => { e.stopPropagation(); panel.style.display = 'flex'; };
        panel.querySelector('#sniff-close').onmousedown = (e) => e.stopPropagation();
        panel.querySelector('#sniff-close').onclick = () => panel.style.display = 'none';

        panel.querySelector('#btn-scan').onclick = () => sniffStatic();
        panel.querySelector('#btn-clear').onclick = () => clearList();

        const monitorBtn = panel.querySelector('#btn-monitor');
        monitorBtn.onclick = () => toggleMonitor(monitorBtn, floater);

        const checkAllBox = panel.querySelector('#check-all');
        checkAllBox.onchange = (e) => {
            const isChecked = e.target.checked;
            document.querySelectorAll('.item-check').forEach(cb => cb.checked = isChecked);
            updateDownloadBtn();
        };

        panel.querySelector('#btn-dl-sel').onclick = () => downloadSelected();
        makeDraggable(panel, panel.querySelector('#sniff-header'));
    }

    // === 核心逻辑：单元素处理 ===
    function processElement(el) {
        if (!el) return;

        // 1. 检查 filesrc 属性 (针对 <span filesrc="...">)
        // 这是用户特别指定的需求
        if (el.getAttribute('filesrc')) {
            const src = el.getAttribute('filesrc');
            const name = el.innerText.trim() || el.textContent.trim();
            checkAndAdd(src, 'FILE_SRC', name);
            return; // 如果命中了 filesrc，通常不需要再检查 href/src
        }

        // 2. 检查 IMG 标签
        if (el.tagName === 'IMG') {
            const name = (el.getAttribute('alt') || el.getAttribute('title') || '').trim();
            checkAndAdd(el.src || el.currentSrc, 'IMAGE', name);
        }

        // 3. 检查 A 标签
        if (el.tagName === 'A') {
            const href = el.href;
            if (href) {
                let name = (el.innerText || el.getAttribute('title') || '').trim();
                // 如果文字无效，尝试去父级找
                if (!name || /^(下载|点击|查看|download)$/i.test(name)) {
                    if (el.parentElement && el.parentElement.tagName !== 'BODY') {
                        name = el.parentElement.innerText.replace(/下载|点击|查看/g, '').trim();
                    }
                }

                const lowerHref = href.toLowerCase();
                if (lowerHref.endsWith('.pdf')) checkAndAdd(href, 'PDF', name);
                else if (lowerHref.endsWith('.doc') || lowerHref.endsWith('.docx')) checkAndAdd(href, 'DOC', name);
                else if (lowerHref.endsWith('.xls') || lowerHref.endsWith('.xlsx')) checkAndAdd(href, 'XLS', name);
                else if (lowerHref.endsWith('.zip') || lowerHref.endsWith('.rar')) checkAndAdd(href, 'ZIP', name);
                else if (isImageUrl(href)) checkAndAdd(href, 'IMAGE_LINK', name);
            }
        }

        // 4. 检查背景图
        if (el.style && el.style.backgroundImage) {
            const bg = el.style.backgroundImage;
            if (bg && bg !== 'none') {
                 const match = bg.match(/url\(['"]?(.*?)['"]?\)/);
                 if (match && match[1] && isImageUrl(match[1])) checkAndAdd(match[1], 'BG_IMG', '');
            }
        }
    }

    // === 核心逻辑：扫描节点及其子树 ===
    function scanNode(node) {
        if (!node || node.nodeType !== 1) return;

        // 1. 处理节点本身
        processElement(node);

        // 2. 处理节点内部的所有相关子元素
        // querySelectorAll 性能远好于手动递归 JS
        // 重点：查找 [filesrc] 属性的元素
        const children = node.querySelectorAll('img, a, [filesrc]');
        children.forEach(child => processElement(child));
    }

    // === 数据存储与去重 ===
    function checkAndAdd(url, type, suggestedName) {
        if (!url || url.startsWith('data:')) return;
        let absoluteUrl;
        try { absoluteUrl = new URL(url, location.href).href; } catch(e) { return; }

        if (seenUrls.has(absoluteUrl)) return;
        seenUrls.add(absoluteUrl);

        let finalName = suggestedName;
        const urlFilename = absoluteUrl.split('/').pop().split('?')[0];

        if (!finalName) finalName = urlFilename || 'unknown_file';

        // 文件名净化
        finalName = finalName.replace(/[\\/:*?"<>|\r\n]/g, '').trim();
        if (finalName.length > 80) finalName = finalName.substring(0, 80);

        // 补全后缀
        const ext = getExtension(absoluteUrl);
        if (ext && !finalName.toLowerCase().endsWith(ext)) {
            finalName += ext;
        }

        const resource = { url: absoluteUrl, type: type, name: finalName };
        foundResources.push(resource);
        addValToUI(resource, foundResources.length - 1);
    }

    function getExtension(url) {
        const cleanUrl = url.split('?')[0];
        const dotIndex = cleanUrl.lastIndexOf('.');
        if (dotIndex !== -1 && (cleanUrl.length - dotIndex) <= 6) {
            return cleanUrl.substring(dotIndex).toLowerCase();
        }
        return '';
    }

    function isImageUrl(url) {
        const clean = url.split('?')[0].toLowerCase();
        return TARGET_EXTENSIONS.some(ext => clean.endsWith(ext) && !['.pdf','.doc','.docx','.xls','.xlsx','.zip','.rar'].includes(ext));
    }

    // === UI 渲染 ===
    function addValToUI(res, index) {
        const container = document.getElementById('sniff-content');
        const emptyTip = document.getElementById('empty-tip');
        if (emptyTip) emptyTip.style.display = 'none';

        const row = document.createElement('div');
        row.className = 'sniff-item';

        // 预览图逻辑
        let previewHtml = '';
        if (res.type === 'FILE_SRC' || ['IMAGE', 'BG_IMG', 'IMAGE_LINK'].includes(res.type)) {
            // 如果 filesrc 指向的是非图片文件(如pdf)，显示文字图标
            if (isImageUrl(res.url)) {
                 previewHtml = `<img src="${res.url}" class="item-preview" onerror="this.style.display='none'">`;
            } else {
                 const ext = getExtension(res.url).replace('.','') || 'FILE';
                 previewHtml = `<div class="item-preview" style="display:flex;align-items:center;justify-content:center;font-size:10px;font-weight:bold;background:#eee;color:#555;">${ext.toUpperCase().substring(0,4)}</div>`;
            }
        } else {
            const extLabel = res.name.split('.').pop() || res.type;
            previewHtml = `<div class="item-preview" style="display:flex;align-items:center;justify-content:center;font-size:10px;font-weight:bold;background:#eee;color:#555;">${extLabel.toUpperCase().substring(0,4)}</div>`;
        }

        row.innerHTML = `
            <input type="checkbox" class="item-check" data-idx="${index}">
            ${previewHtml}
            <div class="item-info">
                <span class="item-name" title="${res.name}">${res.name}</span>
                <a href="${res.url}" target="_blank" class="item-url" title="${res.url}">${res.url}</a>
            </div>
            <button class="mini-btn btn-copy">复制</button>
        `;

        row.querySelector('.btn-copy').onclick = (e) => {
            GM_setClipboard(res.url);
            e.target.innerText = 'OK';
            setTimeout(() => e.target.innerText = '复制', 1000);
        };

        row.querySelector('.item-check').onchange = () => {
            updateDownloadBtn();
            if (!row.querySelector('.item-check').checked) document.getElementById('check-all').checked = false;
        };

        row.onclick = (e) => {
            if (e.target.tagName !== 'BUTTON' && e.target.tagName !== 'A' && e.target.type !== 'checkbox') {
                const cb = row.querySelector('.item-check');
                cb.checked = !cb.checked;
                cb.dispatchEvent(new Event('change'));
            }
        };

        container.insertBefore(row, container.firstChild);
    }

    function downloadSelected() {
        const checkedBoxes = document.querySelectorAll('.item-check:checked');
        if (checkedBoxes.length === 0) return;
        if (!confirm(`确认下载选中的 ${checkedBoxes.length} 个文件?`)) return;

        checkedBoxes.forEach((cb, i) => {
            const idx = cb.getAttribute('data-idx');
            const resource = foundResources[idx];
            setTimeout(() => {
                GM_download({
                    url: resource.url,
                    name: resource.name,
                    saveAs: false
                });
            }, i * 500);
        });
    }

    function clearList() {
        foundResources = [];
        seenUrls.clear();
        document.getElementById('sniff-content').innerHTML = `<div id="empty-tip" style="text-align:center;color:#999;margin-top:50px;"><p>列表已清空</p></div>`;
        document.getElementById('check-all').checked = false;
        updateDownloadBtn();
    }

    function updateDownloadBtn() {
        const count = document.querySelectorAll('.item-check:checked').length;
        const btn = document.getElementById('btn-dl-sel');
        if (count > 0) {
            btn.style.display = 'inline-block';
            btn.innerHTML = `📥 下载选中 (${count})`;
        } else {
            btn.style.display = 'none';
        }
    }

    function makeDraggable(element, handle) {
        let pos1=0, pos2=0, pos3=0, pos4=0;
        handle.onmousedown = (e) => {
            e.preventDefault(); pos3 = e.clientX; pos4 = e.clientY; element.style.transform = "none";
            document.onmouseup = () => { document.onmouseup=null; document.onmousemove=null; };
            document.onmousemove = (e) => {
                e.preventDefault(); pos1 = pos3 - e.clientX; pos2 = pos4 - e.clientY; pos3 = e.clientX; pos4 = e.clientY;
                element.style.top = (element.offsetTop - pos2) + "px"; element.style.left = (element.offsetLeft - pos1) + "px";
            };
        };
    }

    function toggleMonitor(btn, floater) {
        isObserving = !isObserving;
        if (isObserving) {
            btn.innerHTML = "⏹ 停止"; btn.classList.add('active'); floater.classList.add('recording');
            startObserver();
        } else {
            btn.innerHTML = "🔴 监听"; btn.classList.remove('active'); floater.classList.remove('recording');
            stopObserver();
        }
    }

    function startObserver() {
        if(observer) return;
        // 配置 MutationObserver
        observer = new MutationObserver((mutations) => {
            for(let m of mutations) {
                // 如果是新增节点 (childList)
                if(m.type==='childList') {
                    m.addedNodes.forEach(n => {
                        // 确保是元素节点
                        if(n.nodeType===1) {
                            scanNode(n);
                        }
                    });
                }
                // 如果是属性变化 (attributes)
                // 某些情况下 filesrc 是后续加上去的
                else if(m.type==='attributes') {
                    // 如果变化的是我们关心的属性
                    if (['src', 'href', 'filesrc'].includes(m.attributeName)) {
                        processElement(m.target);
                    }
                }
            }
        });

        observer.observe(document.body, {
            childList: true,
            subtree: true,
            attributes: true,
            attributeFilter: ['src', 'href', 'filesrc'] // 监听这些属性的变化
        });
    }

    function stopObserver() { if(observer){observer.disconnect(); observer=null;} }

    function sniffStatic() { scanNode(document.body); }

    window.addEventListener('load', createUI);
})();

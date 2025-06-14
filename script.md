// ==UserScript==
// @name         Vuetorrent Webhook Trigger
// @namespace    http://tampermonkey.net/
// @version      1.5
// @description  Add right-click webhook action in Vuetorrent, CORS-free using GM_xmlhttpRequest
// @match        http://YOUR-TORRENT-CLIENT:PORT/*
// @grant        GM_xmlhttpRequest
// ==/UserScript==

(function () {
    'use strict';

    console.log("🟢 Vuetorrent Webhook script loaded");

    let lastClickedHash = null;

    function getInfohashFromRow(row) {
        const headers = Array.from(document.querySelectorAll("thead th")).map(th => th.innerText.trim());
        const index = headers.findIndex(h => h.toLowerCase().includes("infohash"));
        if (index === -1) {
            console.warn("❌ Infohash column not found");
            return null;
        }

        const cell = row.querySelectorAll("td")[index];
        if (!cell) return null;

        const hash = cell.innerText.trim();
        console.log("📦 Found infohash:", hash);
        return hash;
    }

    // Capture right-clicks on torrent rows
    document.addEventListener("contextmenu", (e) => {
        const row = e.target.closest("tr");
        if (!row) return;
        lastClickedHash = getInfohashFromRow(row);
    });

    const observer = new MutationObserver(() => {
        const menu = document.querySelector(".v-overlay__content .v-list");
        if (!menu || menu.querySelector("#trigger-webhook")) return;
        if (!lastClickedHash) {
            console.warn("⚠️ No infohash found on menu open");
            return;
        }

        const item = document.createElement("div");
        item.id = "trigger-webhook";
        item.className = "v-list-item v-list-item--link v-theme--dark-legacy v-list-item--density-default v-list-item--one-line rounded-0 v-list-item--variant-text px-3";
        item.style.cursor = "pointer";
        item.innerHTML = `
            <span class="v-list-item__overlay"></span>
            <span class="v-list-item__underlay"></span>
            <div class="v-list-item__content" data-no-activator="">
                <div class="d-flex">
                    <i class="mdi-wifi mdi v-icon notranslate v-theme--dark-legacy v-icon--size-default mr-2" aria-hidden="true"></i>
                    <span>Cross Seed</span>
                    <div class="v-spacer"></div>
                </div>
            </div>
        `;

        item.onclick = () => {
            const hash = lastClickedHash;
            console.log("📡 Triggering webhook for", hash);

            GM_xmlhttpRequest({
                method: "POST",
                url: `http://YOUR-SERVER:PORT/api/webhook?apikey=CROSS-SEED-API-KEY`,
                headers: { "Content-Type": "application/x-www-form-urlencoded" },
                data: `infoHash=${hash}`,
                onload: function (res) {
                    alert(`✅ Webhook sent! Status: ${res.status}`);
                },
                onerror: function (err) {
                    alert("❌ Webhook failed");
                    console.error(err);
                }
            });

            lastClickedHash = null;
        };

        menu.appendChild(item);
        console.log("✅ Webhook menu item added");
    });

    observer.observe(document.body, { childList: true, subtree: true });
})();

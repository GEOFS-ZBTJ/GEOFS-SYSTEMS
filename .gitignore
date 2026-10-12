
// ==UserScript==
// @name         GeoFS Cold Start Procedure Panel
// @namespace    http://tampermonkey.net/
// @version      1.0
// @description  为 GeoFS 添加一个模拟冷舱启动程序的交互面板，包含引擎启动逻辑和极限着陆检查单风格。
// @author       Assistant
// @match        *://*.geofs.io/*
// @match        *://geofs.io/*
// @grant        none
// @run-at       document-idle
// ==/UserScript==

(function() {
    'use strict';

    // 配置常量
    const CONFIG = {
        panelWidth: '320px',
        panelHeight: '500px',
        primaryColor: '#0ea5e9', // Sky blue
        dangerColor: '#ef4444',  // Red
        successColor: '#22c55e', // Green
        bgColor: 'rgba(15, 23, 42, 0.95)', // Slate 900 with opacity
        textColor: '#f8fafc'
    };

    // 启动步骤定义
    const STARTUP_STEPS = [
        { id: 'battery', label: '主电池 (Master Battery)', status: 'off', required: true },
        { id: 'avionics', label: '航电系统 (Avionics)', status: 'off', required: true },
        { id: 'beacon', label: '信标灯 (Beacon Light)', status: 'off', required: false },
        { id: 'fuel_pump', label: '燃油泵 (Fuel Pump)', status: 'off', required: true },
        { id: 'mixture', label: '混合比 (Mixture)', status: 'cut', required: true }, // cut, rich
        { id: 'throttle', label: '油门 (Throttle)', status: 'closed', required: true }, // closed, open
        { id: 'starter', label: '启动机 (Starter)', status: 'off', action: 'trigger' }, // Action button
        { id: 'engine', label: '引擎状态 (Engine)', status: 'stopped', readonly: true }
    ];

    let panelState = JSON.parse(JSON.stringify(STARTUP_STEPS));
    let isPanelVisible = false;

    // 初始化面板
    function initPanel() {
        if (document.getElementById('geofs-coldstart-panel')) return;

        const panel = document.createElement('div');
        panel.id = 'geofs-coldstart-panel';
        panel.style.cssText = `
            position: fixed;
            top: 20px;
            right: 20px;
            width: ${CONFIG.panelWidth};
            height: auto;
            max-height: ${CONFIG.panelHeight};
            background-color: ${CONFIG.bgColor};
            color: ${CONFIG.textColor};
            border: 1px solid ${CONFIG.primaryColor};
            border-radius: 8px;
            box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
            z-index: 99999;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            font-size: 14px;
            display: flex;
            flex-direction: column;
            overflow: hidden;
            transition: opacity 0.3s ease;
            backdrop-filter: blur(4px);
        `;

        // 头部
        const header = document.createElement('div');
        header.style.cssText = `
            padding: 10px 15px;
            background: linear-gradient(90deg, rgba(14,165,233,0.2) 0%, rgba(15,23,42,0) 100%);
            border-bottom: 1px solid rgba(255,255,255,0.1);
            display: flex;
            justify-content: space-between;
            align-items: center;
            cursor: move;
        `;
        header.innerHTML = `
            <span style="font-weight: bold; color: ${CONFIG.primaryColor};">✈️ 冷舱启动程序</span>
            <button id="cs-minimize" style="background:none; border:none; color:#fff; cursor:pointer;">_</button>
        `;

        // 内容区域
        const content = document.createElement('div');
        content.id = 'cs-content';
        content.style.cssText = `
            padding: 15px;
            overflow-y: auto;
            flex-grow: 1;
        `;

        // 底部状态栏
        const footer = document.createElement('div');
        footer.style.cssText = `
            padding: 10px;
            background: rgba(0,0,0,0.3);
            text-align: center;
            font-size: 12px;
            color: #94a3b8;
        `;
        footer.innerHTML = '<span id="cs-status-text">准备就绪</span>';

        panel.appendChild(header);
        panel.appendChild(content);
        panel.appendChild(footer);
        document.body.appendChild(panel);

        renderSteps();
        setupDrag(panel, header);
        
        // 最小化按钮逻辑
        document.getElementById('cs-minimize').addEventListener('click', () => {
            content.style.display = content.style.display === 'none' ? 'block' : 'none';
            footer.style.display = footer.style.display === 'none' ? 'block' : 'none';
        });
    }

    // 渲染步骤列表
    function renderSteps() {
        const content = document.getElementById('cs-content');
        if (!content) return;
        content.innerHTML = '';

        panelState.forEach((step, index) => {
            const row = document.createElement('div');
            row.style.cssText = `
                display: flex;
                justify-content: space-between;
                align-items: center;
                margin-bottom: 12px;
                padding-bottom: 8px;
                border-bottom: 1px solid rgba(255,255,255,0.05);
            `;

            const label = document.createElement('span');
            label.textContent = step.label;
            label.style.flex = '1';

            const control = document.createElement('div');
            
            if (step.readonly) {
                // 只读状态显示
                const statusBadge = document.createElement('span');
                statusBadge.textContent = step.status === 'running' ? 'RUNNING' : 'STOPPED';
                statusBadge.style.cssText = `
                    padding: 2px 8px;
                    border-radius: 4px;
                    font-size: 11px;
                    font-weight: bold;
                    background-color: ${step.status === 'running' ? CONFIG.successColor : '#475569'};
                    color: white;
                `;
                control.appendChild(statusBadge);
            } else if (step.action === 'trigger') {
                // 动作按钮
                const btn = document.createElement('button');
                btn.textContent = step.status === 'active' ? '启动中...' : '启动引擎';
                btn.disabled = step.status === 'active';
                btn.style.cssText = `
                    background-color: ${CONFIG.primaryColor};
                    color: white;
                    border: none;
                    padding: 4px 10px;
                    border-radius: 4px;
                    cursor: pointer;
                    font-size: 12px;
                    transition: background 0.2s;
                `;
                btn.onmouseover = () => btn.style.backgroundColor = '#0284c7';
                btn.onmouseout = () => btn.style.backgroundColor = CONFIG.primaryColor;
                btn.onclick = () => handleEngineStart();
                control.appendChild(btn);
            } else {
                // 开关切换
                const toggle = document.createElement('button');
                const isOn = step.status === 'on' || step.status === 'rich' || step.status === 'open';
                toggle.textContent = isOn ? 'ON' : 'OFF';
                toggle.style.cssText = `
                    background-color: ${isOn 
};
})();

---
layout: page
title: My Tools
date: 2023-10-28
categories: [Projects, GitHub]
tags: [tools, github, jekyll]
icon: fa-solid fa-terminal
order: 1
---

Below are some of the tools I've developed and hosted on GitHub.

<div class="tool-box">
    <h3><a href="https://github.com/RhinoSecurityLabs/pacu/blob/master/pacu/modules/ebs__enum_snapshots_unauth/main.py">ebs__enum_snapshots_unauth Pacu Module </a></h3>
    <p>A Pacu module for unauthenticated reconnaissance of public EBS snapshots, based on a keyword, account ID, or a list of keywords or account IDs.</p>
    <img src="/assets/img/tools/ebs__enum_snapshots_unauth.png" alt="Preview" />
</div>

<div class="tool-box">
    <h3><a href="https://github.com/Y4nush/PassedRoleLambdaExfiltrator">PassedRoleLambdaExfiltrator</a></h3>
    <p>An AWS offensive security tool designed to detect and exploit the iam:PassRole with Lambda for privilege escalation. It enumerates vulnerable roles, deploys a function, and sends credentials to an ngrok endpoint using an HTTP POST request by invoking the malicious lambda function.</p>
    <img src="/assets/img/tools/PassedRoleLambdaExfiltrator.png" alt="Preview" />
</div>

<div class="tool-box">
    <h3><a href="https://github.com/Tetrisponse/DMARCARE">DMARCARE</a></h3>
    <p>DMARCARE is a Python tool that extracts and analyzes DMARC records from specified domains, providing insights on potential spoofing threats.</p>
    <img src="/assets/img/tools/dmarcare.png" alt="Preview" />
</div>

<style>
/* Light mode styles */
.tool-box {
    border: 1px solid #ddd;
    padding: 20px;
    margin-bottom: 20px;
    background-color: #1b1b1e;
}
/* Dark mode styles */
@media (prefers-color-scheme: dark) {
    .tool-box {
        border-color: #444;
        background-color: #1b1b1e;
        color: #ddd;
    }
    .tool-box a {
        color: #1e90ff;
    }
}
.tool-box h3 {
    margin-top: 0;
}
.tool-box img {
    max-width: 100%;
    height: auto;
    display: block;
    margin: 0 auto;
}
</style>

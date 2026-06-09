# Novel-Encrypt 小说批量加密工厂

纯前端的 TXT 小说批量加密 / 解密工具。所有处理都在浏览器本地完成，**文件和密码不会上传到任何服务器**。

| 页面 | 用途 |
| --- | --- |
| `index.html` | 批量加密：选择多个 `.txt`，输出 `xxx_enc.txt` 打包成 ZIP |
| `decrypt.html` | 批量解密：选择多个 `xxx_enc.txt`，还原 `xxx.txt` 打包成 ZIP |

## 使用方法

1. 打开页面（GitHub Pages 在线访问，或直接双击本地 HTML 文件）。
2. 输入全局密码（所有文件共用同一个密码，**密码丢失后无法找回**）。
3. 多选 `.txt` 文件，点击按钮，浏览器会自动下载 ZIP 压缩包。
4. 解压后将 `_enc.txt` 文件上传到 GitHub 等托管处即可。

> 打包下载依赖 CDN 上的 JSZip，加载页面时需要联网。

## 文本编码支持

读取文件时会自动识别编码，避免中文乱码：

- UTF-8（含 BOM）
- UTF-16 LE / BE（带 BOM，常见于 Windows 记事本）
- GBK / GB2312（按 GB18030 解码）

解密输出统一为 UTF-8。

## 加密格式（v2）

每个加密文件是一行纯 ASCII 文本：

```
NOVELENC2:<盐 base64>:<IV base64>:<密文+校验标签 base64>
```

- 算法：AES-256-GCM（Web Crypto API，带完整性校验，密码错误会明确报错）
- 密钥派生：PBKDF2-SHA256，600,000 次迭代，16 字节随机盐
- IV：每个文件独立的 12 字节随机值
- 同一批次的文件共用一个随机盐（密钥只需派生一次，批量处理快），盐写在每个文件头部，单个文件可独立解密

### 在阅读器中集成解密

任何网页阅读器都可以用下面这段无依赖的代码解密 v2 格式：

```js
async function decryptNovel(content, password) {
  const b64 = s => Uint8Array.from(atob(s), c => c.charCodeAt(0));
  const parts = content.trim().split(':');
  if (parts[0] !== 'NOVELENC2' || parts.length !== 4) throw new Error('不是 NOVELENC2 格式');
  const baseKey = await crypto.subtle.importKey(
    'raw', new TextEncoder().encode(password), 'PBKDF2', false, ['deriveKey']);
  const key = await crypto.subtle.deriveKey(
    { name: 'PBKDF2', salt: b64(parts[1]), iterations: 600000, hash: 'SHA-256' },
    baseKey, { name: 'AES-GCM', length: 256 }, false, ['decrypt']);
  const plain = await crypto.subtle.decrypt({ name: 'AES-GCM', iv: b64(parts[2]) }, key, b64(parts[3]));
  return new TextDecoder().decode(plain);
}
```

## 旧版文件兼容

早期版本使用 `CryptoJS.AES.encrypt(text, password)`（OpenSSL 兼容格式，内容以 `U2FsdGVkX1` 开头）。

- `decrypt.html` **同时支持新旧两种格式**，旧文件无需重新加密即可解密。
- 新版加密输出与旧格式**不兼容**：如果你有基于 `CryptoJS.AES.decrypt` 的外部阅读器，请按上面的代码片段升级，或继续用 `decrypt.html` 解密。
- 升级原因：旧格式的密钥派生基于 MD5（OpenSSL EVP_BytesToKey）且无完整性校验；新格式使用 PBKDF2 + AES-GCM，抗暴力破解更强，且能可靠检测密码错误。

## 安全说明

- 浏览器需支持 Web Crypto API：通过 HTTPS 访问（GitHub Pages 即可）或直接打开本地文件均可，普通 HTTP 站点不行。
- 前端加密的强度取决于密码本身，请使用足够长且不易猜测的密码。
- 这是静态页面，没有后端，不存储任何数据。

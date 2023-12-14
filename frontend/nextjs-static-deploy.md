# 前端开发实践

## Nextjs delopy 静态页面时无法本地打开

1. Nextjs静态部署的方式：
在`next.config.js` 中增加` output: 'export'`：

```
/**
 * @type {import('next').NextConfig}
 */
const nextConfig = {
  output: 'export',
 
  // Optional: Change links `/me` -> `/me/` and emit `/me.html` -> `/me/index.html`
  // trailingSlash: true,
 
  // Optional: Prevent automatic `/me` -> `/me/`, instead preserve `href`
  // skipTrailingSlashRedirect: true,
 
  // Optional: Change the output directory `out` -> `dist`
  // distDir: 'dist',
}
 
module.exports = nextConfig
```

2. 将`build`命令改成`next build && next export`


打包后的产物，在本地打开会报2个错误，一个是跨域问题，一个是`/_next/xxxx`无法找到或打开。
解决跨域问题：将生成的`/out/index.html`中js标签中所有的`crossorigin=""`删除

解决`/_next/xxxx`无法找到的问题: [Generated static files html files have wrong assets paths](https://github.com/vercel/next.js/issues/8158)  
将所有`/_next/xxxx` 改成`./_next/xxxx`


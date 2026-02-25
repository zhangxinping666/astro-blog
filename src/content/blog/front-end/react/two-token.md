---
title: 后台管理系统登录模块（双token的实现思路）
link: two-token
catalog: true
date: 2025-01-05 13:00:00
description: 后台管理系统登录模块, 详细双token的实现思路
tags:
  - React
  - this
categories:
  - [前端, React]
---

这里为您将提供的内容整理成了一份结构清晰、排版优雅的 Markdown 文档。我严格遵守了“不删减内容”的原则，同时修复了原文中因为复制粘贴导致的换行和注释错乱问题，并为代码块添加了 TypeScript 语法高亮，方便您直接阅读或复制使用。

***

# React + TypeScript 后台管理：登录模块与 Axios 双 Token 封装实现

最近在写后台管理，这里分享一下我的登录模块的实现。我是使用 React + TypeScript 实现的，主要是登录的逻辑和双 Token 的处理方式，以及请求接口的二次封装 Axios。

**1. 首先我们需要渲染登录界面的窗口**
这个很简单就不详细讲解了，然后主要就是关于点击登录按钮的接口的调用。

**封装我们的接口（封装是非常有必要的）：**
​编辑 *(注：此处原文可能包含图片位置标记)*

下面是我们的登录接口，然后 `request` 就是我二次封装的 axios：

```typescript
export function login(data: LoginData) {
  return request.post<LoginResult>('/backstage/login', data);
}
```

这里详细讲解一下关于 axios 的封装。对于每一个要写项目的时候，只要有后端请求接口，我们都需要封装 axios，这一步很重要。下面是对 axios 封装的主要实现，主要是基于双 token 的 axios 二次封装请求。

---

## 创建 axios 的封装核心文件（request）讲解

### 1. 双 token 的实现逻辑？

#### 1.1 长短 token 是什么
当用户登录成功之后会返回一个 json 数据，里面有两个 token，一个是短 token（`access_token`），一个是长 token（`refresh_token`）。
* **access_token** 是访问令牌。因为在请求具有权限接口的时候需要请求头，里面需要放用户 token，这个请求头里面的 token 就是我们的 `access_token`。`access_token` 的存在时间很短。
* **refresh_token** 是刷新令牌，用于生成短 token。

#### 1.2 双 token 的更新逻辑
当 `access_token` 过期时，需要使用 `refresh_token` 来生成一个新的 `access_token`。那么什么时候会触发这个刷新机制呢？
其实就是当调用权限接口的时候，如果 `access_token` 过期了，服务器就会返回一个 `401 unauthorized`。前端的响应拦截器会获取所有的 api 请求，当它获取到 401 的时候，就知道 `access_token` 过期了，然后就刷新 token 了。

当获取到了 `access_token` 之后，需要存储 `access_token`，这是为了确保后续所有新发起的 API 请求都能使用这个最新的 `access_token`。之前失败的请求并没有被直接抛弃，而是被暂存到了一个“待重试队列”（`failedQueue`）中，等待重新发送新的请求。

---

### 2. axios 二次封装实现思路

首先引入我们的核心库，第一行就不进行解释了；第二行就是自己封装的一些存储、获取、清除 token 的方法，见名知意。

```typescript
import axios, { AxiosInstance, AxiosRequestConfig, AxiosResponse, AxiosError } from 'axios';
import { getAccessToken, getRefreshToken, setTokens, clearTokens } from './token';
```

#### 2.1 创建队列和函数
* **isRefreshing**：用于确保在多个请求同时 401 时，只有一个 token 刷新请求被发送，防止重复刷新。
* **failedQueue 队列**：当 `isRefreshing` 为 true 时（表示已经有刷新 token 的请求在进行中），所有后续因 401 失败的请求都会被推入 `failedQueue`。这些请求会返回一个新的 Promise，等待 token 刷新成功后被解决。
* **processQueue 函数**：负责在 token 刷新成功或失败后，处理 `failedQueue` 中的所有请求。如果成功，用新的 token 重新发送；如果失败，则拒绝这些请求。

```typescript
let isRefreshing: boolean = false; // 存储因 token 过期而失败的请求队列

// 定义队列中每个元素的类型
interface FailedRequest {
  resolve: (value?: string | PromiseLike<string>) => void;
  reject: (reason?: any) => void;
}

let failedQueue: FailedRequest[] = [];

const processQueue = (error: Error | null, token: string | null = null): void => {
  failedQueue.forEach(prom => {
    if (error) {
      prom.reject(error);
    } else {
      prom.resolve(token as string); // 确保在没有错误时 token 不为 null
    }
  });
  failedQueue = [];
};
```

#### 2.2 创建 axios 实例
使用 `axios.create()` 创建一个独立的 axios 实例，避免污染全局 axios。

```typescript
const request: AxiosInstance = axios.create({
  // 在 .env 文件中配置的请求配置
  baseURL: 'api基础路径', 
  timeout: 10000, // 请求超时时间
});
```

#### 2.3 创建请求拦截器
* 动态添加 `access_token`，从存储位置获取 `access_token`，并将其添加到请求头 `Authorization` 中。通常格式为 `Bearer ${token}`。
* 除了上面的请求头，还可以添加其他请求头，如 `Content-Type`。
* 可以实现全局的请求 Loading 动画。

```typescript
// --- 请求拦截器 ---
request.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    const accessToken = getAccessToken();
    if (accessToken) {
      // 在请求头中添加 Authorization 字段
      if (!config.headers) {
        config.headers = new axios.AxiosHeaders();
      }
      config.headers['Authorization'] = `Bearer ${accessToken}`;
    }
    return config;
  },
  (error: AxiosError) => {
    return Promise.reject(error);
  },
);
```

#### 2.4 创建响应拦截器
* 后端返回的数据通常会包裹在 data 当中，可以直接返回 `response.data`。
* 处理 HTTP 状态码非 2xx 的错误。
* **无感刷新 token (核心)：**
    1.  **捕获 401 Unauthorized 错误**：这是最关键的一步。当后端因为 `access_token` 过期而返回 401 时，拦截此错误。
    2.  **调用刷新接口**：使用 `refresh_token` 去请求新的 `access_token`。
    3.  **重发失败请求**：获取到新的 `access_token` 后，将刚才失败的请求（`error.config`）用新 token 重新发送一次。
    4.  **并发请求处理**：当多个请求同时因为 token 过期而失败时，要确保刷新 token 的接口只被调用一次。后续失败的请求应被“暂存”，等待新 token 获取后再统一重发。

```typescript
// --- 响应拦截器 ---
request.interceptors.response.use(
  // 响应成功 (HTTP 状态码为 2xx)
  (response: AxiosResponse<any>) => {
    // 通常后端会把数据包裹在 data 中，这里直接返回 data，简化业务代码
    return response.data;
  }, 
  // 响应失败 (HTTP 状态码非 2xx)
  async (error: AxiosError) => {
    const originalRequest = error.config as
      | (InternalAxiosRequestConfig & { _retry?: boolean })
      | undefined;

    // 如果没有config，直接返回错误
    if (!originalRequest) {
      console.error('Request Error: No config available');
      return Promise.reject(error);
    } 
    
    // 检查是否是 401 Unauthorized 错误，并且不是刷新 token 的请求本身
    if (error.response?.status === 401 && !originalRequest._retry) {
      // 如果正在刷新 token，则将当前失败的请求加入队列
      if (isRefreshing) {
        return new Promise<string>((resolve, reject) => {
          failedQueue.push({
            resolve: (value?: string | PromiseLike<string>) => resolve(value as string),
            reject,
          });
        })
          .then((token) => {
            if (!originalRequest.headers) {
              originalRequest.headers = new axios.AxiosHeaders();
            }
            originalRequest.headers['Authorization'] = `Bearer ${token}`;
            return request(originalRequest); // 使用新 token 重新发送请求
          })
          .catch((err) => {
            return Promise.reject(err);
          });
      }
      
      originalRequest._retry = true; // 标记此请求已尝试过重试
      isRefreshing = true;

      const refreshToken = getRefreshToken();
      if (!refreshToken) {
        // 如果没有 refresh_token，直接跳转到登录页
        console.error('No refresh token available.');
        clearTokens(); 
        // window.location.href = '/login'; // 或使用 router.push('/login')
        return Promise.reject(new Error('No refresh token, redirect to login.'));
      }

      try {
        // --- 调用刷新 Token 的 API ---
        // 注意：这里需要使用一个不带拦截器的 axios 实例来发请求，避免循环调用
        const response = await axios.post<{
          data: {
            access_token: string;
            refresh_token: string;
          };
        }>('登录接口api', {
          refresh_token: refreshToken,
        });

        const { access_token: newAccessToken, refresh_token: newRefreshToken } = response.data.data; 
        
        // 1. 更新本地存储的 token
        setTokens(newAccessToken, newRefreshToken); 
        
        // 2. 处理并重发等待队列中的请求
        processQueue(null, newAccessToken); 
        
        // 3. 重发本次失败的请求
        if (!originalRequest.headers) {
          originalRequest.headers = new axios.AxiosHeaders();
        }
        originalRequest.headers['Authorization'] = `Bearer ${newAccessToken}`;
        return request(originalRequest);
        
      } catch (refreshError: unknown) {
        // 刷新 token 失败，清除所有 token 并重定向到登录页
        console.error('Failed to refresh token:', refreshError);
        clearTokens();
        processQueue(refreshError as Error, null); 
        // window.location.href = '/login'; // 或使用 router.push('/login')
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    } 
    
    // 对于其他错误，直接抛出
    // 处理错误信息，确保类型安全
    const errorMessage =
      error.response?.data &&
      typeof error.response.data === 'object' &&
      'message' in error.response.data
        ? (error.response.data as { message: string }).message
        : error.message;
    console.error('Request Error:', errorMessage);
    return Promise.reject(error);
  },
);
```

#### 2.5 完整的 axios 二次封装（request）

```typescript
import axios, { AxiosInstance, InternalAxiosRequestConfig, AxiosResponse, AxiosError } from 'axios';
import { getAccessToken, getRefreshToken, setTokens, clearTokens } from '../stores/token';

// --- 状态变量 ---
// 标记是否正在刷新 token，防止重复刷新
let isRefreshing: boolean = false;

// 存储因 token 过期而失败的请求队列
// 定义队列中每个元素的类型
interface FailedRequest {
  resolve: (value?: string | PromiseLike<string>) => void;
  reject: (reason?: unknown) => void;
}

let failedQueue: FailedRequest[] = [];

/**
 * @description 处理队列中的请求
 * @param {Error | null} error - 刷新 token 过程中的错误
 * @param {string | null} token - 新的 access_token
 */
const processQueue = (error: Error | null, token: string | null = null): void => {
  failedQueue.forEach((prom) => {
    if (error) {
      prom.reject(error);
    } else {
      prom.resolve(token as string); // 确保在没有错误时 token 不为 null
    }
  });
  failedQueue = [];
};

// --- 创建 Axios 实例 ---
const request: AxiosInstance = axios.create({
  // 在 .env 文件中配置的请求配置
  baseURL: '基础api',
  timeout: 10000, // 请求超时时间
});

// --- 请求拦截器 ---
request.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    const accessToken = getAccessToken();
    if (accessToken) {
      // 在请求头中添加 Authorization 字段
      if (!config.headers) {
        config.headers = new axios.AxiosHeaders();
      }
      config.headers['Authorization'] = `Bearer ${accessToken}`;
    }
    return config;
  },
  (error: AxiosError) => {
    return Promise.reject(error);
  },
);

// --- 响应拦截器 ---
request.interceptors.response.use(
  // 响应成功 (HTTP 状态码为 2xx)
  (response: AxiosResponse<any>) => {
    // 通常后端会把数据包裹在 data 中，这里直接返回 data，简化业务代码
    return response.data;
  }, 
  
  // 响应失败 (HTTP 状态码非 2xx)
  async (error: AxiosError) => {
    const originalRequest = error.config as
      | (InternalAxiosRequestConfig & { _retry?: boolean })
      | undefined;

    // 如果没有config，直接返回错误
    if (!originalRequest) {
      console.error('Request Error: No config available');
      return Promise.reject(error);
    } 
    
    // 检查是否是 401 Unauthorized 错误，并且不是刷新 token 的请求本身
    if (error.response?.status === 401 && !originalRequest._retry) {
      // 如果正在刷新 token，则将当前失败的请求加入队列
      if (isRefreshing) {
        return new Promise<string>((resolve, reject) => {
          failedQueue.push({
            resolve: (value?: string | PromiseLike<string>) => resolve(value as string),
            reject,
          });
        })
          .then((token) => {
            if (!originalRequest.headers) {
              originalRequest.headers = new axios.AxiosHeaders();
            }
            originalRequest.headers['Authorization'] = `Bearer ${token}`;
            return request(originalRequest); // 使用新 token 重新发送请求
          })
          .catch((err) => {
            return Promise.reject(err);
          });
      }
      
      originalRequest._retry = true; // 标记此请求已尝试过重试
      isRefreshing = true;

      const refreshToken = getRefreshToken();
      if (!refreshToken) {
        // 如果没有 refresh_token，直接跳转到登录页
        console.error('No refresh token available.');
        clearTokens(); 
        // window.location.href = '/login'; // 或使用 router.push('/login')
        return Promise.reject(new Error('No refresh token, redirect to login.'));
      }

      try {
        // --- 调用刷新 Token 的 API ---
        // 注意：这里需要使用一个不带拦截器的 axios 实例来发请求，避免循环调用
        const response = await axios.post<{
          data: {
            access_token: string;
            refresh_token: string;
          };
        }>('login接口', {
          refresh_token: refreshToken,
        });

        const { access_token: newAccessToken, refresh_token: newRefreshToken } = response.data.data; 
        
        // 1. 更新本地存储的 token
        setTokens(newAccessToken, newRefreshToken); 
        
        // 2. 处理并重发等待队列中的请求
        processQueue(null, newAccessToken); 
        
        // 3. 重发本次失败的请求
        if (!originalRequest.headers) {
          originalRequest.headers = new axios.AxiosHeaders();
        }
        originalRequest.headers['Authorization'] = `Bearer ${newAccessToken}`;
        return request(originalRequest);
        
      } catch (refreshError: unknown) {
        // 刷新 token 失败，清除所有 token 并重定向到登录页
        console.error('Failed to refresh token:', refreshError);
        clearTokens();
        processQueue(refreshError as Error, null); 
        // window.location.href = '/login'; // 或使用 router.push('/login')
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    } 
    
    // 对于其他错误，直接抛出
    // 处理错误信息，确保类型安全
    const errorMessage =
      error.response?.data &&
      typeof error.response.data === 'object' &&
      'message' in error.response.data
        ? (error.response.data as { message: string }).message
        : error.message;
    console.error('Request Error:', errorMessage);
    return Promise.reject(error);
  },
);

export default request;
```
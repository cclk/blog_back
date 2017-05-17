---
title: Direct3d纹理渲染播放视频c++类
date: 2017-05-17 12:01:32
categories:
reward: false
tags:
     - blog
     - 音视频
     - ffmpeg
     - direct3d
---

## 背景
Direct3d纹理渲染播放视频c++类

## 头文件（D3DTexture.h）

<!--more-->

``` cpp
#ifndef d3d_render_h__
#define d3d_render_h__

#include <memory>
#include <windows.h>
#include <d3d9.h>

class D3DTexture
{
public:
	D3DTexture();
	~D3DTexture();

public:
	bool Create(HWND hWnd, size_t width, size_t height);
	void Destroy();

	bool RenderBGRA(uint8_t *rgb, uint32_t width, uint32_t height);

private:
	void ResizeTexture(size_t width, size_t height);
	void SetSamplerState();
	bool TestCooperativeLevel();

private:
	HWND m_hWnd;
	size_t m_width;
	size_t m_height;

	IDirect3D9 *m_d3d;
	IDirect3DDevice9 *m_d3dDevice;
	IDirect3DVertexBuffer9 *m_d3dVertexBuffer;
	IDirect3DTexture9 *m_d3dTexture;
};

#endif // d3d_render_h__
```

## 源文件（D3DTexture.cpp）

``` cpp
#include "D3DTexture.h"

#define D3DFVF_CUSTOMVERTEX (D3DFVF_XYZ|D3DFVF_TEX1)

struct D3dCustomVertex {
	float x, y, z;
	float u, v;
};

D3DTexture::D3DTexture()
	: m_hWnd(NULL)
	, m_width(0)
	, m_height(0)
	, m_d3d(NULL)
	, m_d3dDevice(NULL)
	, m_d3dVertexBuffer(NULL)
	, m_d3dTexture(NULL)
{

}

D3DTexture::~D3DTexture()
{
}

bool D3DTexture::Create(HWND hWnd, size_t width, size_t height)
{
	do
	{
		Destroy();

		if (NULL == hWnd)
		{
			std::cout << "hWnd is null";
			break;
		}
		else
		{
			m_hWnd = hWnd;
		}

		m_d3d = Direct3DCreate9(D3D_SDK_VERSION);
		if (nullptr == m_d3d)
		{
			std::cout << "Direct3DCreate9 faild";
			break;
		}

		D3DPRESENT_PARAMETERS d3d_params = {};
		d3d_params.Windowed = TRUE;
		d3d_params.SwapEffect = D3DSWAPEFFECT_COPY;

		IDirect3DDevice9* d3d_device = NULL;
		HRESULT hRet = m_d3d->CreateDevice(D3DADAPTER_DEFAULT, D3DDEVTYPE_HAL, m_hWnd,
			D3DCREATE_SOFTWARE_VERTEXPROCESSING | D3DCREATE_MULTITHREADED, &d3d_params, &d3d_device);
		if (D3D_OK != hRet)
		{
			std::cout << "CreateDevice faild:" << hRet;
			break;
		}
		m_d3dDevice = d3d_device;

		IDirect3DVertexBuffer9* vertex_buffer = NULL;
		const int kRectVertices = 4;
		hRet = m_d3dDevice->CreateVertexBuffer(kRectVertices * sizeof(D3dCustomVertex), 0, D3DFVF_CUSTOMVERTEX, D3DPOOL_MANAGED, &vertex_buffer, NULL);
		if (D3D_OK != hRet)
		{
			std::cout << "CreateVertexBuffer faild:" << hRet;
			break;
		}
		m_d3dVertexBuffer = vertex_buffer;

		ResizeTexture(width, height);

		m_d3dDevice->Present(NULL, NULL, NULL, NULL);

		return true;
	} while (0);

	Destroy();
	return false;
}

void D3DTexture::Destroy()
{
	if (m_d3dTexture)
	{
		m_d3dTexture->Release();
		m_d3dTexture = NULL;
	}

	if (m_d3dVertexBuffer)
	{
		m_d3dVertexBuffer->Release();
		m_d3dVertexBuffer = NULL;
	}

	if (m_d3dDevice)
	{
		m_d3dDevice->Release();
		m_d3dDevice = NULL;
	}

	if (m_d3d)
	{
		m_d3d->Release();
		m_d3d = NULL;
	}
}

bool D3DTexture::RenderBGRA(uint8_t *rgb, uint32_t width, uint32_t height)
{
	if (width != m_width || height != m_height)
	{
		ResizeTexture(width, height);
	}

	TestCooperativeLevel();

	D3DLOCKED_RECT lock_rect;
	HRESULT hr = m_d3dTexture->LockRect(0, &lock_rect, NULL, 0);
	if (hr != D3D_OK)
	{
		std::cout << "LockRect faild:" << hr;
		return false;
	}

	///to do copy
	char * pDest = reinterpret_cast<char*>(lock_rect.pBits);
	const char * pSrc = reinterpret_cast<const char*>(rgb);
	int stride = lock_rect.Pitch;
	int pixel_w_size = width * 32 / 8;

	for (int i = 0; i < height; i++)
	{
		memcpy(pDest, pSrc, pixel_w_size);
		pDest += stride;
		pSrc += pixel_w_size;
	}

	m_d3dTexture->UnlockRect(0);

	//draw
	m_d3dDevice->Clear(0, NULL, D3DCLEAR_TARGET | D3DCLEAR_ZBUFFER, D3DCOLOR_XRGB(100, 100, 100), 1.0f, 0);
	m_d3dDevice->BeginScene();
	m_d3dDevice->SetFVF(D3DFVF_CUSTOMVERTEX);
	m_d3dDevice->SetStreamSource(0, m_d3dVertexBuffer, 0, sizeof(D3dCustomVertex));
	m_d3dDevice->SetTexture(0, m_d3dTexture);//启用纹理
	m_d3dDevice->DrawPrimitive(D3DPT_TRIANGLESTRIP, 0, 2);//利用索引缓存配合顶点缓存绘制图形
	m_d3dDevice->EndScene();

	m_d3dDevice->Present(NULL, NULL, NULL, NULL);
	return true;
}

void D3DTexture::ResizeTexture(size_t width, size_t height)
{
	if (m_d3dTexture)
	{
		m_d3dTexture->Release();
		m_d3dTexture = NULL;
	}

	m_width = width;
	m_height = height;
	IDirect3DTexture9* texture = NULL;

	m_d3dDevice->CreateTexture(static_cast<UINT>(m_width),
		static_cast<UINT>(m_height),
		1,
		0,
		D3DFMT_A8R8G8B8,
		D3DPOOL_MANAGED,
		&texture,
		NULL);
	m_d3dTexture = texture;

	// Vertices for the video frame to be rendered to.
	static const D3dCustomVertex rect[] = {
		{ -1.0f, -1.0f, 0.0f, 0.0f, 1.0f },
		{ -1.0f, 1.0f, 0.0f, 0.0f, 0.0f },
		{ 1.0f, -1.0f, 0.0f, 1.0f, 1.0f },
		{ 1.0f, 1.0f, 0.0f, 1.0f, 0.0f },
	};

	void* buf_data;
	if (m_d3dVertexBuffer->Lock(0, 0, &buf_data, 0) != D3D_OK)
	{
		return;
	}

	memcpy(buf_data, &rect, sizeof(rect));
	m_d3dVertexBuffer->Unlock();

	SetSamplerState();
}

void D3DTexture::SetSamplerState()
{
	IDirect3DDevice9_SetVertexShader(m_d3dDevice, NULL);
	IDirect3DDevice9_SetFVF(m_d3dDevice, D3DFVF_XYZ | D3DFVF_DIFFUSE | D3DFVF_TEX1);
	IDirect3DDevice9_SetRenderState(m_d3dDevice, D3DRS_ZENABLE, D3DZB_FALSE);
	IDirect3DDevice9_SetRenderState(m_d3dDevice, D3DRS_CULLMODE, D3DCULL_NONE);
	IDirect3DDevice9_SetRenderState(m_d3dDevice, D3DRS_LIGHTING, FALSE);

	/* Enable color modulation by diffuse color */
	IDirect3DDevice9_SetTextureStageState(m_d3dDevice, 0, D3DTSS_COLOROP, D3DTOP_MODULATE);
	IDirect3DDevice9_SetTextureStageState(m_d3dDevice, 0, D3DTSS_COLORARG1, D3DTA_TEXTURE);
	IDirect3DDevice9_SetTextureStageState(m_d3dDevice, 0, D3DTSS_COLORARG2, D3DTA_DIFFUSE);

	/* Enable alpha modulation by diffuse alpha */
	IDirect3DDevice9_SetTextureStageState(m_d3dDevice, 0, D3DTSS_ALPHAOP, D3DTOP_MODULATE);
	IDirect3DDevice9_SetTextureStageState(m_d3dDevice, 0, D3DTSS_ALPHAARG1, D3DTA_TEXTURE);
	IDirect3DDevice9_SetTextureStageState(m_d3dDevice, 0, D3DTSS_ALPHAARG2, D3DTA_DIFFUSE);

	/* Enable separate alpha blend function, if possible */
	IDirect3DDevice9_SetRenderState(m_d3dDevice, D3DRS_SEPARATEALPHABLENDENABLE, TRUE);

	/* Disable second texture stage, since we're done */
	IDirect3DDevice9_SetTextureStageState(m_d3dDevice, 1, D3DTSS_COLOROP, D3DTOP_DISABLE);
	IDirect3DDevice9_SetTextureStageState(m_d3dDevice, 1, D3DTSS_ALPHAOP, D3DTOP_DISABLE);

	/* Set an identity world and view matrix */
	D3DMATRIX matrix;
	matrix.m[0][0] = 1.0f;
	matrix.m[0][1] = 0.0f;
	matrix.m[0][2] = 0.0f;
	matrix.m[0][3] = 0.0f;
	matrix.m[1][0] = 0.0f;
	matrix.m[1][1] = 1.0f;
	matrix.m[1][2] = 0.0f;
	matrix.m[1][3] = 0.0f;
	matrix.m[2][0] = 0.0f;
	matrix.m[2][1] = 0.0f;
	matrix.m[2][2] = 1.0f;
	matrix.m[2][3] = 0.0f;
	matrix.m[3][0] = 0.0f;
	matrix.m[3][1] = 0.0f;
	matrix.m[3][2] = 0.0f;
	matrix.m[3][3] = 1.0f;
	IDirect3DDevice9_SetTransform(m_d3dDevice, D3DTS_WORLD, &matrix);
	IDirect3DDevice9_SetTransform(m_d3dDevice, D3DTS_VIEW, &matrix);

	//设置Mipmap纹理采样器参数
	int SamplerIdx = 0;
	m_d3dDevice->SetTexture(SamplerIdx, m_d3dTexture);
	m_d3dDevice->SetSamplerState(SamplerIdx, D3DSAMP_ADDRESSU, D3DTADDRESS_CLAMP);
	m_d3dDevice->SetSamplerState(SamplerIdx, D3DSAMP_ADDRESSV, D3DTADDRESS_MIRROR);
	m_d3dDevice->SetSamplerState(SamplerIdx, D3DSAMP_ADDRESSW, D3DTADDRESS_WRAP);
	m_d3dDevice->SetSamplerState(SamplerIdx, D3DSAMP_MAGFILTER, D3DTEXF_LINEAR);
	m_d3dDevice->SetSamplerState(SamplerIdx, D3DSAMP_MINFILTER, D3DTEXF_ANISOTROPIC);
	m_d3dDevice->SetSamplerState(SamplerIdx, D3DSAMP_MIPFILTER, D3DTEXF_POINT);
	m_d3dDevice->SetSamplerState(SamplerIdx, D3DSAMP_MIPMAPLODBIAS, 1);// Mipmap层级向高精度偏移1个层级
	m_d3dDevice->SetSamplerState(SamplerIdx, D3DSAMP_MAXANISOTROPY, 4);// 最大异性采样阈值设置为4
}

bool D3DTexture::TestCooperativeLevel()
{
	if (NULL == m_d3dDevice)
	{
		return false;
	}

	HRESULT hState = m_d3dDevice->TestCooperativeLevel();

	switch (hState)
	{
	case D3DERR_DEVICELOST://设备丢失
		return false;

	case D3DERR_DEVICENOTRESET://设备可以恢复
		return Create(m_hWnd, m_width, m_height);

	case  D3D_OK:
		break;

	default:
		return false;
	}

	return true;
}

```
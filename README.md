# dx12-discord-hook
```c++
/////////////////////
// D3D12 HOOK ImGui//
/////////////////////

#include "d3d12hook.h"
#include "menu/theme/style.hpp"
#include "menu/menu.hpp"
#include "../../modules/loop.hpp"
#include "../../modules/recoil/recoil.hpp"
#include "elements/elements.hpp"
#include "../../../ext/saftyhook/safetyhook.hpp"
#include "../../../ext/callstack/CallStack-Spoofer.h"
#include "../../sdk/structs/ida.h"
#include "../../../ext/Syskey/Syskey.h"

int countnum = -1;
bool nopants_enabled = true;

//
// Discord types
//

// 
// Probblay the best way to make an internal overlay thats not shit for dx12 icl
//

typedef HRESULT(__fastcall* DiscordHookFunction_t)(_QWORD* originalFunctionPtr, __int64 hookName, void* targetFunction, void* hookFunction);


//=========================================================================================================================//

typedef HRESULT(APIENTRY* Present12) (IDXGISwapChain* pSwapChain, UINT SyncInterval, UINT Flags);
Present12 oPresent = NULL;

typedef void(APIENTRY* DrawInstanced)(ID3D12GraphicsCommandList* dCommandList, UINT VertexCountPerInstance, UINT InstanceCount, UINT StartVertexLocation, UINT StartInstanceLocation);
DrawInstanced oDrawInstanced = NULL;

typedef void(APIENTRY* DrawIndexedInstanced)(ID3D12GraphicsCommandList* dCommandList, UINT IndexCountPerInstance, UINT InstanceCount, UINT StartIndexLocation, INT BaseVertexLocation, UINT StartInstanceLocation);
DrawIndexedInstanced oDrawIndexedInstanced = NULL;

typedef void(APIENTRY* ExecuteCommandLists)(ID3D12CommandQueue* queue, UINT NumCommandLists, ID3D12CommandList* ppCommandLists);
ExecuteCommandLists oExecuteCommandLists = NULL;

//=========================================================================================================================//

bool ShowMenu = true;
bool ImGui_Initialised = false;

namespace Process {
	DWORD ID;
	HANDLE Handle;
	HWND Hwnd;
	HMODULE Module;
	WNDPROC WndProc;
	int WindowWidth;
	int WindowHeight;
	LPCSTR Title;
	LPCSTR ClassName;
	LPCSTR Path;
}

namespace DirectX12Interface {
	ID3D12Device* Device = nullptr;
	ID3D12DescriptorHeap* DescriptorHeapBackBuffers;
	ID3D12DescriptorHeap* DescriptorHeapImGuiRender;
	ID3D12GraphicsCommandList* CommandList;
	ID3D12CommandQueue* CommandQueue;

	struct _FrameContext {
		ID3D12CommandAllocator* CommandAllocator;
		ID3D12Resource* Resource;
		D3D12_CPU_DESCRIPTOR_HANDLE DescriptorHandle;
	};

	uintx_t BuffersCounts = -1;
	_FrameContext* FrameContext;
}

//=========================================================================================================================//

extern IMGUI_IMPL_API LRESULT ImGui_ImplWin32_WndProcHandler(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam);
LRESULT APIENTRY WndProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam) {
	if (ShowMenu) {
		ImGui_ImplWin32_WndProcHandler(hwnd, uMsg, wParam, lParam);
		return true;
	}
	return CallWindowProc(Process::WndProc, hwnd, uMsg, wParam, lParam);
}

//=========================================================================================================================//
SafetyHookMid g_hkPresent{};

struct hkcontext {
	IDXGISwapChain3* rcx;
	UINT rdx;
	UINT r8;
};

hkcontext hkPresent(IDXGISwapChain3* pSwapChain, UINT SyncInterval, UINT Flags) {

	if (!ImGui_Initialised) {
		if (SUCCEEDED(pSwapChain->GetDevice(__uuidof(ID3D12Device), (void**)&DirectX12Interface::Device))) {
			ImGui::CreateContext();

			ImGuiIO& io = ImGui::GetIO(); (void)io;
			//io.Fonts->AddFontFromFileTTF("C:/Windows/Fonts/msyh.ttc", 18.f, nullptr, io.Fonts->GetGlyphRangesChineseFull());
			ImGui::GetIO().WantCaptureMouse || ImGui::GetIO().WantTextInput || ImGui::GetIO().WantCaptureKeyboard;
			io.ConfigFlags |= ImGuiConfigFlags_NavEnableKeyboard;

			DXGI_SWAP_CHAIN_DESC Desc;
			pSwapChain->GetDesc(&Desc);
			Desc.Flags = DXGI_SWAP_CHAIN_FLAG_ALLOW_MODE_SWITCH;
			Desc.OutputWindow = Process::Hwnd;
			Desc.Windowed = ((GetWindowLongPtr(Process::Hwnd, GWL_STYLE) & WS_POPUP) != 0) ? false : true;

			DirectX12Interface::BuffersCounts = Desc.BufferCount;
			DirectX12Interface::FrameContext = new DirectX12Interface::_FrameContext[DirectX12Interface::BuffersCounts];

			D3D12_DESCRIPTOR_HEAP_DESC DescriptorImGuiRender = {};
			DescriptorImGuiRender.Type = D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV;
			DescriptorImGuiRender.NumDescriptors = DirectX12Interface::BuffersCounts;
			DescriptorImGuiRender.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;

			if (DirectX12Interface::Device->CreateDescriptorHeap(&DescriptorImGuiRender, IID_PPV_ARGS(&DirectX12Interface::DescriptorHeapImGuiRender)) != S_OK)
				return { pSwapChain, SyncInterval, Flags };

			ID3D12CommandAllocator* Allocator;
			if (DirectX12Interface::Device->CreateCommandAllocator(D3D12_COMMAND_LIST_TYPE_DIRECT, IID_PPV_ARGS(&Allocator)) != S_OK)
				return { pSwapChain, SyncInterval, Flags };

			for (size_t i = 0; i < DirectX12Interface::BuffersCounts; i++) {
				DirectX12Interface::FrameContext[i].CommandAllocator = Allocator;
			}

			if (DirectX12Interface::Device->CreateCommandList(0, D3D12_COMMAND_LIST_TYPE_DIRECT, Allocator, NULL, IID_PPV_ARGS(&DirectX12Interface::CommandList)) != S_OK ||
				DirectX12Interface::CommandList->Close() != S_OK)
				return { pSwapChain, SyncInterval, Flags };

			D3D12_DESCRIPTOR_HEAP_DESC DescriptorBackBuffers;
			DescriptorBackBuffers.Type = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
			DescriptorBackBuffers.NumDescriptors = DirectX12Interface::BuffersCounts;
			DescriptorBackBuffers.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;
			DescriptorBackBuffers.NodeMask = 1;

			if (DirectX12Interface::Device->CreateDescriptorHeap(&DescriptorBackBuffers, IID_PPV_ARGS(&DirectX12Interface::DescriptorHeapBackBuffers)) != S_OK)
				return { pSwapChain, SyncInterval, Flags };

			const auto RTVDescriptorSize = DirectX12Interface::Device->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_RTV);
			D3D12_CPU_DESCRIPTOR_HANDLE RTVHandle = DirectX12Interface::DescriptorHeapBackBuffers->GetCPUDescriptorHandleForHeapStart();

			for (size_t i = 0; i < DirectX12Interface::BuffersCounts; i++) {
				ID3D12Resource* pBackBuffer = nullptr;
				DirectX12Interface::FrameContext[i].DescriptorHandle = RTVHandle;
				pSwapChain->GetBuffer(i, IID_PPV_ARGS(&pBackBuffer));
				DirectX12Interface::Device->CreateRenderTargetView(pBackBuffer, nullptr, RTVHandle);
				DirectX12Interface::FrameContext[i].Resource = pBackBuffer;
				RTVHandle.ptr += RTVDescriptorSize;
			}

			ImGui_ImplWin32_Init(Process::Hwnd);
			ImGui_ImplDX12_Init(DirectX12Interface::Device, DirectX12Interface::BuffersCounts, DXGI_FORMAT_R8G8B8A8_UNORM, DirectX12Interface::DescriptorHeapImGuiRender, DirectX12Interface::DescriptorHeapImGuiRender->GetCPUDescriptorHandleForHeapStart(), DirectX12Interface::DescriptorHeapImGuiRender->GetGPUDescriptorHandleForHeapStart());
			ImGui_ImplDX12_CreateDeviceObjects();
			//ImGui::GetIO().ImeWindowHandle = Process::Hwnd;
			Process::WndProc = (WNDPROC)SetWindowLongPtr(Process::Hwnd, GWLP_WNDPROC, (__int3264)(LONG_PTR)WndProc);

			ImGui::StyleColorsDark();
			theme::init::Style();

			DXGI_SWAP_CHAIN_DESC1 swapChainDesc;
			pSwapChain->GetDesc1(&swapChainDesc);
		}
		ImGui_Initialised = true;
	}

	if (DirectX12Interface::CommandQueue == nullptr)
		return { pSwapChain, SyncInterval, Flags };


	if (SPOOF_CALL(nt::GetKey)(VK_INSERT) & 1) ShowMenu = !ShowMenu;
	ImGui_ImplDX12_NewFrame();
	ImGui_ImplWin32_NewFrame();
	ImGui::NewFrame();

	ImGui::GetIO().MouseDrawCursor = ShowMenu;
	if (ShowMenu == true) {
		menu();
	}

	elements::watermark();

	modules::loop();

	modules::recoil::applyRecoil();

	ImGui::EndFrame();

	DirectX12Interface::_FrameContext& CurrentFrameContext = DirectX12Interface::FrameContext[pSwapChain->GetCurrentBackBufferIndex()];
	CurrentFrameContext.CommandAllocator->Reset();

	D3D12_RESOURCE_BARRIER Barrier;
	Barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
	Barrier.Flags = D3D12_RESOURCE_BARRIER_FLAG_NONE;
	Barrier.Transition.pResource = CurrentFrameContext.Resource;
	Barrier.Transition.Subresource = D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES;
	Barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_PRESENT;
	Barrier.Transition.StateAfter = D3D12_RESOURCE_STATE_RENDER_TARGET;

	DirectX12Interface::CommandList->Reset(CurrentFrameContext.CommandAllocator, nullptr);
	DirectX12Interface::CommandList->ResourceBarrier(1, &Barrier);
	DirectX12Interface::CommandList->OMSetRenderTargets(1, &CurrentFrameContext.DescriptorHandle, FALSE, nullptr);
	DirectX12Interface::CommandList->SetDescriptorHeaps(1, &DirectX12Interface::DescriptorHeapImGuiRender);

	ImGui::Render();
	ImGui_ImplDX12_RenderDrawData(ImGui::GetDrawData(), DirectX12Interface::CommandList);
	Barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_RENDER_TARGET;
	Barrier.Transition.StateAfter = D3D12_RESOURCE_STATE_PRESENT;
	DirectX12Interface::CommandList->ResourceBarrier(1, &Barrier);
	DirectX12Interface::CommandList->Close();
	DirectX12Interface::CommandQueue->ExecuteCommandLists(1, reinterpret_cast<ID3D12CommandList* const*>(&DirectX12Interface::CommandList));
	return { pSwapChain, SyncInterval, Flags };
}

void hkPresentWarpper(safetyhook::Context& ctx) {


	hkcontext newRegs = hkPresent((IDXGISwapChain3*)ctx.rcx, (UINT)ctx.rdx, (UINT)ctx.r8);

	ctx.rcx = (uintptr_t)newRegs.rcx;
	ctx.rdx = (uintptr_t)newRegs.rdx;
	ctx.r8 = (uintptr_t)newRegs.r8;
}


SafetyHookMid g_hkExecuteCommandLists{};
//void hkExecuteCommandLists(ID3D12CommandQueue* queue, UINT NumCommandLists, ID3D12CommandList* ppCommandLists) {
void hkExecuteCommandLists(safetyhook::Context& ctx) {
	if (!DirectX12Interface::CommandQueue)
		DirectX12Interface::CommandQueue = (ID3D12CommandQueue*)ctx.rcx;

	//oExecuteCommandLists(queue, NumCommandLists, ppCommandLists);
}

//=========================================================================================================================//

void supermagic() {
	while (true) {
		try {
			modules::outlineLoop();
			Sleep(3);
		}
		catch (...) {}
	}
}

void mainlogic() {
	while (true) {
		HANDLE hThread = CreateThread(0, 0, (LPTHREAD_START_ROUTINE)supermagic, 0, 0, 0);
		if (hThread) {
			WaitForSingleObject(hThread, INFINITE); // Wait for the thread to finish
			CloseHandle(hThread);  // Clean up thread handle
		}
	}
}

typedef int(APIENTRY* discordhook1) (_QWORD* originalFunctionPtr, __int64 hookName, int* targetFunction, __int64 hookFunction);
discordhook1 discordhook = NULL;

int __fastcall hook(_QWORD* originalFunctionPtr, __int64 hookName, int* targetFunction, __int64 hookFunction) {

	//std::cout << originalFunctionPtr << std::endl;
	//std::cout << hookName << std::endl;
	//std::cout << targetFunction << std::endl;

	if (!discordhook) {
		std::cout << "[-] discordhook is NULL!" << std::endl;
		return 0;
	}

	int a = discordhook(originalFunctionPtr, hookName, targetFunction, hookFunction);

	std::cout << "[" << a << "] hookFunction: " << hookFunction << std::endl;

	return a;
}

void InitD3D12Hook() {

	//CreateThread(0, 0, (LPTHREAD_START_ROUTINE)mainlogic, 0, 0, 0);

	auto start_time = std::chrono::steady_clock::now();

	while (true) {
		if (SPOOF_CALL(nt::GetKey)(VK_INSERT) & 0x8000) { // Check if Insert key is pressed
			ShowMenu = true;
			//std::cout << "[+] Insert key pressed! Continuing execution...\n";
			break;
		}
		
		auto elapsed_time = std::chrono::steady_clock::now() - start_time;
		if (std::chrono::duration_cast<std::chrono::seconds>(elapsed_time).count() >= 60) {
			ShowMenu = false;
			//std::cout << "[-] Timeout reached (60s). Continuing execution...\n";
			break;
		}

		Sleep(10); // Prevent CPU overuse
	}

	bool WindowFocus = false;
	while (WindowFocus == false) {
		DWORD ForegroundWindowProcessID;
		GetWindowThreadProcessId(GetForegroundWindow(), &ForegroundWindowProcessID);
		if (GetCurrentProcessId() == ForegroundWindowProcessID) {

			Process::ID = GetCurrentProcessId();
			if (Process::ID == 0)
				continue;

			Process::Handle = GetCurrentProcess();
			if (Process::Handle == NULL)
				continue;

			Process::Hwnd = GetForegroundWindow();
			if (Process::Hwnd == NULL)
				continue;

			RECT TempRect;
			GetWindowRect(Process::Hwnd, &TempRect);
			Process::WindowWidth = TempRect.right - TempRect.left;
			Process::WindowHeight = TempRect.bottom - TempRect.top;

			char TempTitle[MAX_PATH];
			GetWindowTextA(Process::Hwnd, TempTitle, sizeof(TempTitle));
			Process::Title = TempTitle;

			char TempClassName[MAX_PATH];
			GetClassNameA(Process::Hwnd, TempClassName, sizeof(TempClassName));
			Process::ClassName = TempClassName;

			char TempPath[MAX_PATH];
			GetModuleFileNameExA(Process::Handle, NULL, TempPath, sizeof(TempPath));
			Process::Path = TempPath;

			WindowFocus = true;
		}
	}
	bool InitHook = false;
	while (InitHook == false) {
		if (DirectX12::Init() == true) {
			//CreateHook(54, (void**)&oExecuteCommandLists, hkExecuteCommandLists);

			//void* _54 = CreateHook(140);
			//g_hookD3D11Present = safetyhook::create_mid(_54, hookD3D11PresentWarpper);
			//oPresent = (Present12)_54;

			//void* _140 = CreateHook(54);
			//g_hkExecuteCommandLists = safetyhook::create_mid(_140, hookhkExecuteCommandListsWarpper);

			g_hkExecuteCommandLists = safetyhook::create_mid(CreateHook(54), hkExecuteCommandLists);

			auto start_time = std::chrono::steady_clock::now();
			uintptr_t discordbase = (uintptr_t)memory::get_safe_module(_x(L"DiscordHook64.dll"));
			while (discordbase == 0) {
				discordbase = (uintptr_t)memory::get_safe_module(_x(L"DiscordHook64.dll"));

				auto elapsed = std::chrono::steady_clock::now() - start_time;
				if (std::chrono::duration_cast<std::chrono::seconds>(elapsed).count() >= 30) {
					MessageBoxA(nullptr,
						_x("Failed to find DiscordHook64.dll\n\nPLEASE MAKE SURE YOU HAVE DISCORD OPEN & THE OVERLAY IS ON [1]"),
						_x("Error"),
						MB_OK | MB_ICONERROR | MB_TOPMOST);
					exit(0);
				}

				std::this_thread::sleep_for(std::chrono::milliseconds(100));
			}

			//uintptr_t discordHookingFunction = discordbase + 0x62E0;

			//if (MH_CreateHook((LPVOID)discordHookingFunction, hook, (LPVOID*)&discordhook) != MH_OK || MH_EnableHook((LPVOID)discordHookingFunction) != MH_OK) {
			//	std::cout << "[-] Failed to create hook!" << std::endl;
			//}

			if (dev::debug) std::cout << _x("[+] DiscordHook64.dll found at: 0x") << std::hex << discordbase << std::endl;

			//0x167d0
			//55 41 57 41 56 56 57 53 48 83 EC ? 48 8D 6C 24 ? 44 89 C6
			//e9 ? ? ? ? 56 57 53
			//56 57 53 48 83 ec ? 48 8d 6c 24 ? 44 89 c6
			//uintptr_t discordPresent = discordbase + 0x167d0;
			uintptr_t discordPresent = (uintptr_t)memory::sig((void*)discordbase, _("e9 ? ? ? ? 56 57 53"));
			if (!discordPresent)
			{
				MessageBoxA(nullptr, _("Faild to find Discord Present \n\nPLEASE MAKE SURE U HAVE DISCORD OPEN & THE OVERLAY IS ON [2]"), _("Error"), MB_OK | MB_ICONERROR | MB_OK | MB_TOPMOST);
				if (dev::debug) std::cout << _x("[-] Failed to find Discord Present!") << std::endl;
				return;
			}
			if (dev::debug) std::cout << _x("[+] Discord Present found at: 0x") << std::hex << discordPresent << std::endl;

			discordPresent = memory::follow_jumps(discordPresent);

			if (dev::debug) std::cout << _x("[+] Discord Present resolved at: 0x") << std::hex << discordPresent << std::endl;

			
			//uintptr_t discordHookingFunction = (uintptr_t)memory::sig((void*)discordbase, _("56 48 83 EC ? 4C 89 C0 48 89 CE"));
			//std::cout << "[+] Discord Hooking Function found at: 0x" << std::hex << discordHookingFunction << std::endl;

			//DiscordHookFunction_t DiscordHookFunction = reinterpret_cast<DiscordHookFunction_t>(discordHookingFunction);

			/*
			_QWORD tempoPresent;

			HRESULT result = DiscordHookFunction(
				&tempoPresent,
				(__int64)"IDXGISwapChain::Present",
				reinterpret_cast<void*>(discordPresent),  // Function to hook
				reinterpret_cast<void*>(&hkPresent)// Store original function here
			);

			oPresent = (Present12)tempoPresent;
			*/

			g_hkPresent = safetyhook::create_mid(discordPresent, hkPresentWarpper);

			//if (MH_CreateHook((LPVOID)discordPresent, hkPresent, (LPVOID*)&oPresent) != MH_OK || MH_EnableHook((LPVOID)discordPresent) != MH_OK) {
			//	std::cout << "[-] Failed to create hook!" << std::endl;
			//}
			if (dev::debug) std::cout << _x("[+] DX12 Hook created!") << std::endl;
			//CreateHook(140, (void**)&oPresent, hkPresent);

			//CreateHook(84, (void**)&oDrawInstanced, hkDrawInstanced);
			//CreateHook(85, (void**)&oDrawIndexedInstanced, hkDrawIndexedInstanced);

			InitHook = true;


			while (true) {
				try {
					modules::outlineLoop();
					Sleep(3);
				}
				catch (...) {}
			}
		}
	}
	return;
}

//=========================================================================================================================//

//D3D12 Methods Table:
//[0]   QueryInterface
//[1]   AddRef
//[2]   Release
//[3]   GetPrivateData
//[4]   SetPrivateData
//[5]   SetPrivateDataInterface
//[6]   SetName
//[7]   GetNodeCount
//[8]   CreateCommandQueue
//[9]   CreateCommandAllocator
//[10]  CreateGraphicsPipelineState
//[11]  CreateComputePipelineState
//[12]  CreateCommandList
//[13]  CheckFeatureSupport
//[14]  CreateDescriptorHeap
//[15]  GetDescriptorHandleIncrementSize
//[16]  CreateRootSignature
//[17]  CreateConstantBufferView
//[18]  CreateShaderResourceView
//[19]  CreateUnorderedAccessView
//[20]  CreateRenderTargetView
//[21]  CreateDepthStencilView
//[22]  CreateSampler
//[23]  CopyDescriptors
//[24]  CopyDescriptorsSimple
//[25]  GetResourceAllocationInfo
//[26]  GetCustomHeapProperties
//[27]  CreateCommittedResource
//[28]  CreateHeap
//[29]  CreatePlacedResource
//[30]  CreateReservedResource
//[31]  CreateSharedHandle
//[32]  OpenSharedHandle
//[33]  OpenSharedHandleByName
//[34]  MakeResident
//[35]  Evict
//[36]  CreateFence
//[37]  GetDeviceRemovedReason
//[38]  GetCopyableFootprints
//[39]  CreateQueryHeap
//[40]  SetStablePowerState
//[41]  CreateCommandSignature
//[42]  GetResourceTiling
//[43]  GetAdapterLuid
//[44]  QueryInterface
//[45]  AddRef
//[46]  Release
//[47]  GetPrivateData
//[48]  SetPrivateData
//[49]  SetPrivateDataInterface
//[50]  SetName
//[51]  GetDevice
//[52]  UpdateTileMappings
//[53]  CopyTileMappings
//[54]  ExecuteCommandLists
//[55]  SetMarker
//[56]  BeginEvent
//[57]  EndEvent
//[58]  Signal
//[59]  Wait
//[60]  GetTimestampFrequency
//[61]  GetClockCalibration
//[62]  GetDesc
//[63]  QueryInterface
//[64]  AddRef
//[65]  Release
//[66]  GetPrivateData
//[67]  SetPrivateData
//[68]  SetPrivateDataInterface
//[69]  SetName
//[70]  GetDevice
//[71]  Reset
//[72]  QueryInterface
//[73]  AddRef
//[74]  Release
//[75]  GetPrivateData
//[76]  SetPrivateData
//[77]  SetPrivateDataInterface
//[78]  SetName
//[79]  GetDevice
//[80]  GetType
//[81]  Close
//[82]  Reset
//[83]  ClearState
//[84]  DrawInstanced
//[85]  DrawIndexedInstanced
//[86]  Dispatch
//[87]  CopyBufferRegion
//[88]  CopyTextureRegion
//[89]  CopyResource
//[90]  CopyTiles
//[91]  ResolveSubresource
//[92]  IASetPrimitiveTopology
//[93]  RSSetViewports
//[94]  RSSetScissorRects
//[95]  OMSetBlendFactor
//[96]  OMSetStencilRef
//[97]  SetPipelineState
//[98]  ResourceBarrier
//[99]  ExecuteBundle
//[100] SetDescriptorHeaps
//[101] SetComputeRootSignature
//[102] SetGraphicsRootSignature
//[103] SetComputeRootDescriptorTable
//[104] SetGraphicsRootDescriptorTable
//[105] SetComputeRoot32BitConstant
//[106] SetGraphicsRoot32BitConstant
//[107] SetComputeRoot32BitConstants
//[108] SetGraphicsRoot32BitConstants
//[109] SetComputeRootConstantBufferView
//[110] SetGraphicsRootConstantBufferView
//[111] SetComputeRootShaderResourceView
//[112] SetGraphicsRootShaderResourceView
//[113] SetComputeRootUnorderedAccessView
//[114] SetGraphicsRootUnorderedAccessView
//[115] IASetIndexBuffer
//[116] IASetVertexBuffers
//[117] SOSetTargets
//[118] OMSetRenderTargets
//[119] ClearDepthStencilView
//[120] ClearRenderTargetView
//[121] ClearUnorderedAccessViewUint
//[122] ClearUnorderedAccessViewFloat
//[123] DiscardResource
//[124] BeginQuery
//[125] EndQuery
//[126] ResolveQueryData
//[127] SetPredication
//[128] SetMarker
//[129] BeginEvent
//[130] EndEvent
//[131] ExecuteIndirect
//[132] QueryInterface
//[133] AddRef
//[134] Release
//[135] SetPrivateData
//[136] SetPrivateDataInterface
//[137] GetPrivateData
//[138] GetParent
//[139] GetDevice
//[140] Present
//[141] GetBuffer
//[142] SetFullscreenState
//[143] GetFullscreenState
//[144] GetDesc
//[145] ResizeBuffers
//[146] ResizeTarget
//[147] GetContainingOutput
//[148] GetFrameStatistics
//[149] GetLastPresentCount```

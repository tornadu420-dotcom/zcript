--[[
	MENU DE UTILIDADES - 31 OPÇÕES (LocalScript / Luau)
	Funciona em PC e celular (toque). Nenhuma opção de voo.

	PERSONAGEM (5):     WalkSpeed, JumpPower, Rotação automática, Respawn rápido, Resetar personagem
	CÂMERA (6):         FOV, Zoom máximo, Zoom mínimo, Sensib. da câmera, Sensib. do mouse, Shift Lock
	JOGADORES (4):      Nomes, Distância, Destacar jogadores, Contador de jogadores
	TIMES (1):          Detectar Times (NOVO)
	DESEMPENHO (3):     FPS, Ping, Coordenadas
	UTILIDADES (1):     Anti-AFK
	INTERFACE (9):      Interface ON/OFF, Minimizar, Arrastar, Transparência, Tamanho, Tema,
	                    Animações, Som, Notificações
	CONFIGURAÇÕES (2):  Restaurar configurações, Salvar configurações

	Atalho no PC: RightControl mostra/esconde o menu.
	No celular: use o botão "MENU" que aparece quando a interface está desligada.
]]

--==================================================================
-- Limpeza de execução anterior (evita menus duplicados)
--==================================================================
if shared.MenuUtil30_Cleanup then
	pcall(shared.MenuUtil30_Cleanup)
	shared.MenuUtil30_Cleanup = nil
end

--==================================================================
-- Serviços
--==================================================================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local SoundService = game:GetService("SoundService")
local Stats = game:GetService("Stats")
local VirtualUser = game:GetService("VirtualUser")
local TeamsService = game:GetService("Teams")

local LocalPlayer = Players.LocalPlayer
if not LocalPlayer then
	Players:GetPropertyChangedSignal("LocalPlayer"):Wait()
	LocalPlayer = Players.LocalPlayer
end

--==================================================================
-- Utilidades básicas
--==================================================================
local conns = {}
local function connect(signal, fn)
	local c = signal:Connect(fn)
	table.insert(conns, c)
	return c
end

local function create(class, props, children)
	local inst = Instance.new(class)
	for k, v in pairs(props) do
		if k ~= "Parent" then
			inst[k] = v
		end
	end
	if children then
		for _, c in ipairs(children) do
			c.Parent = inst
		end
	end
	if props.Parent then
		inst.Parent = props.Parent
	end
	return inst
end

local function corner(parent, radius)
	return create("UICorner", { CornerRadius = UDim.new(0, radius), Parent = parent })
end

local orderCounter = 0
local function nextOrder()
	orderCounter += 1
	return orderCounter
end

local function getHum()
	local c = LocalPlayer.Character
	return c and c:FindFirstChildOfClass("Humanoid")
end

local function getRoot()
	local c = LocalPlayer.Character
	return c and c:FindFirstChild("HumanoidRootPart")
end

--==================================================================
-- ScreenGui
--==================================================================
local function getGuiParent()
	local ok, res = pcall(function()
		if typeof(gethui) == "function" then
			return gethui()
		end
		return nil
	end)
	if ok and res then
		return res
	end
	return LocalPlayer:WaitForChild("PlayerGui")
end

local guiParent = getGuiParent()
local oldGui = guiParent:FindFirstChild("MenuUtilidades30")
if oldGui then
	oldGui:Destroy()
end

local gui = create("ScreenGui", {
	Name = "MenuUtilidades30",
	ResetOnSpawn = false,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
	DisplayOrder = 999,
	IgnoreGuiInset = false,
	Parent = guiParent,
})

--==================================================================
-- Valores padrão (lidos do jogo) e configurações atuais
--==================================================================
local waited = 0
while not getHum() and waited < 8 do
	task.wait(0.25)
	waited += 0.25
end

local UGS
pcall(function()
	UGS = UserSettings():GetService("UserGameSettings")
end)

local function readNum(getter, fallback)
	local ok, v = pcall(getter)
	if ok and typeof(v) == "number" then
		return v
	end
	return fallback
end

local D = {
	-- Personagem
	WalkSpeed = 16,
	JumpPower = 50,
	AutoRotate = true,
	RespawnFast = false,
	-- Câmera
	FOV = readNum(function() return workspace.CurrentCamera.FieldOfView end, 70),
	MaxZoom = readNum(function() return LocalPlayer.CameraMaxZoomDistance end, 128),
	MinZoom = readNum(function() return LocalPlayer.CameraMinZoomDistance end, 0.5),
	CamSens = readNum(function() return UGS.GamepadCameraSensitivity end, 1),
	MouseSens = readNum(function() return UGS.MouseSensitivity end, 1),
	ShiftLock = false,
	-- Jogadores
	ShowNames = false,
	ShowDistance = false,
	Highlight = false,
	PlayerCount = false,
	-- Times
	DetectTeams = false,
	-- Desempenho
	ShowFPS = false,
	ShowPing = false,
	ShowCoords = false,
	-- Utilidades
	AntiAFK = false,
	-- Movimento / Fly
	FlyEnabled = false,
	FlyFrozen = false,
	FlySpeed = 100,
	Noclip = false,
	SavedPosition = nil,
	-- Menu rápido
	Quick_Fly = true,
	Quick_Noclip = false,
	Quick_ESP = true,
	Quick_SavePosition = false,
	Quick_GotoPosition = true,
	Quick_VerTimes = false,
	Quick_ShowNames = false,
	Quick_ShowDistance = false,
	Quick_Highlight = false,
	QuickOrder = {"Fly","Noclip","ESP","SavePosition","GotoPosition","VerTimes","ShowNames","ShowDistance","Highlight"},
	-- Interface
	InterfaceOn = true,
	Minimized = false,
	Draggable = true,
	Transparency = 0.05,
	Scale = 1,
	Theme = "Escuro",
	Animations = true,
	Sound = true,
	Notifications = true,
}

local origUseJump = true
do
	local h0 = getHum()
	if h0 then
		D.WalkSpeed = h0.WalkSpeed
		D.JumpPower = h0.JumpPower
		origUseJump = h0.UseJumpPower
	end
end

local S = table.clone(D)

-- Arquivo de configuração
local FILE = "MenuUtilidades30_Config.json"
local HAS_FS = typeof(writefile) == "function" and typeof(readfile) == "function" and typeof(isfile) == "function"

local function readSaved()
	local raw
	if HAS_FS then
		local ok, res = pcall(function()
			if isfile(FILE) then
				return readfile(FILE)
			end
			return nil
		end)
		if ok then
			raw = res
		end
	end
	if not raw and typeof(shared.MenuUtil30_Saved) == "string" then
		raw = shared.MenuUtil30_Saved
	end
	if not raw then
		return nil
	end
	local ok, data = pcall(function()
		return HttpService:JSONDecode(raw)
	end)
	if ok and typeof(data) == "table" then
		return data
	end
	return nil
end

do
	local saved = readSaved()
	if saved then
		for k, v in pairs(saved) do
			if D[k] ~= nil and typeof(v) == typeof(D[k]) then
				S[k] = v
			end
		end
	end
	S.InterfaceOn = true
end

--==================================================================
-- Temas
--==================================================================
local function rgb(r, g, b)
	return Color3.fromRGB(r, g, b)
end

local THEMES = {
	["Escuro"] = { bg = rgb(22, 24, 30), panel = rgb(34, 37, 46), title = rgb(28, 31, 39), text = rgb(236, 239, 245), sub = rgb(146, 154, 172), accent = rgb(80, 140, 255), off = rgb(72, 77, 92) },
	["Claro"] = { bg = rgb(238, 240, 245), panel = rgb(255, 255, 255), title = rgb(222, 227, 237), text = rgb(28, 32, 42), sub = rgb(100, 108, 125), accent = rgb(46, 110, 240), off = rgb(190, 196, 208) },
	["Azul"] = { bg = rgb(12, 24, 48), panel = rgb(20, 38, 72), title = rgb(14, 30, 60), text = rgb(230, 240, 255), sub = rgb(130, 160, 205), accent = rgb(60, 170, 255), off = rgb(50, 72, 110) },
	["Roxo"] = { bg = rgb(26, 18, 42), panel = rgb(42, 30, 66), title = rgb(32, 22, 52), text = rgb(240, 232, 255), sub = rgb(165, 148, 200), accent = rgb(170, 100, 255), off = rgb(74, 60, 105) },
	["Verde"] = { bg = rgb(14, 28, 22), panel = rgb(22, 44, 34), title = rgb(16, 34, 26), text = rgb(230, 250, 240), sub = rgb(130, 178, 152), accent = rgb(60, 210, 130), off = rgb(50, 80, 64) },
}
local THEME_ORDER = { "Escuro", "Claro", "Azul", "Roxo", "Verde" }

if not THEMES[S.Theme] then
	S.Theme = D.Theme
end

local function T()
	return THEMES[S.Theme] or THEMES["Escuro"]
end

local themed = {}
local function reg(inst, prop, key)
	table.insert(themed, { inst, prop, key })
	inst[prop] = T()[key]
	return inst
end

local function applyTheme()
	local th = T()
	for _, e in ipairs(themed) do
		pcall(function()
			e[1][e[2]] = th[e[3]]
		end)
	end
end

local transp = {}
local function regTransp(inst)
	table.insert(transp, inst)
end

local function applyTransparency()
	for _, inst in ipairs(transp) do
		inst.BackgroundTransparency = S.Transparency
	end
end

--==================================================================
-- Animação e som
--==================================================================
local function tween(inst, props, time)
	if S.Animations then
		local tw = TweenService:Create(inst, TweenInfo.new(time or 0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), props)
		tw:Play()
		return tw
	end
	for k, v in pairs(props) do
		inst[k] = v
	end
	return nil
end

local clickSound = create("Sound", {
	Name = "MenuUtil30_Click",
	SoundId = "rbxasset://sounds/electronicpingshort.wav",
	Volume = 0.4,
	Parent = SoundService,
})

local function playClick()
	if S.Sound then
		pcall(function()
			clickSound:Play()
		end)
	end
end

--==================================================================
-- Estado compartilhado da UI
--==================================================================
local handlers = {}
local refreshers = {}
local toggleNames = {}
local teamsApi = {} -- ponte com o módulo "Detectar Times"
local categoryRegistry = {}

local Main, TitleBar, Body, OpenBtn, MinBtn, uiScale, HudFrame, notifHolder
local hudLabels = {}

local notify
local setSetting
local saveConfig

local function refreshAll(instant)
	for _, f in pairs(refreshers) do
		f(instant)
	end
end

function setSetting(key, value, quiet)
	S[key] = value
	local h = handlers[key]
	if h then
		local ok, err = pcall(h, value)
		if not ok then
			warn("[MenuUtil30] Erro em " .. tostring(key) .. ": " .. tostring(err))
		end
	end
	local r = refreshers[key]
	if r then
		r(false)
	end
	if not quiet and toggleNames[key] then
		notify(toggleNames[key] .. ": " .. (value and "ON" or "OFF"))
	end
end

--==================================================================
-- Notificações
--==================================================================
function notify(text, force)
	if not notifHolder then
		return
	end
	if not (S.Notifications or force) then
		return
	end
	local th = T()
	local anim = S.Animations
	local toast = create("Frame", {
		Size = UDim2.new(1, 0, 0, 32),
		BackgroundColor3 = th.panel,
		BackgroundTransparency = anim and 1 or 0.05,
		BorderSizePixel = 0,
		LayoutOrder = nextOrder(),
		Parent = notifHolder,
	})
	corner(toast, 8)
	local stroke = create("UIStroke", { Color = th.accent, Thickness = 1, Transparency = anim and 1 or 0, Parent = toast })
	local lbl = create("TextLabel", {
		BackgroundTransparency = 1,
		Size = UDim2.fromScale(1, 1),
		Text = text,
		TextColor3 = th.text,
		TextSize = 13,
		Font = Enum.Font.GothamMedium,
		TextTruncate = Enum.TextTruncate.AtEnd,
		TextTransparency = anim and 1 or 0,
		Parent = toast,
	})
	create("UIPadding", { PaddingLeft = UDim.new(0, 10), PaddingRight = UDim.new(0, 10), Parent = lbl })

	if anim then
		tween(toast, { BackgroundTransparency = 0.05 }, 0.18)
		tween(stroke, { Transparency = 0 }, 0.18)
		tween(lbl, { TextTransparency = 0 }, 0.18)
	end

	local toasts = {}
	for _, c in ipairs(notifHolder:GetChildren()) do
		if c:IsA("Frame") then
			table.insert(toasts, c)
		end
	end
	if #toasts > 3 then
		toasts[1]:Destroy()
	end

	task.delay(2.2, function()
		if not toast.Parent then
			return
		end
		if S.Animations then
			tween(toast, { BackgroundTransparency = 1 }, 0.2)
			tween(stroke, { Transparency = 1 }, 0.2)
			tween(lbl, { TextTransparency = 1 }, 0.2)
			task.wait(0.22)
		end
		if toast.Parent then
			toast:Destroy()
		end
	end)
end

--==================================================================
-- Construção da janela principal
--==================================================================
Main = create("Frame", {
	Name = "Main",
	Size = UDim2.fromOffset(330, 420),
	Position = UDim2.fromOffset(12, 12),
	BorderSizePixel = 0,
	ClipsDescendants = true,
	Active = true,
	Parent = gui,
})
reg(Main, "BackgroundColor3", "bg")
regTransp(Main)
corner(Main, 10)
reg(create("UIStroke", { Thickness = 1, Parent = Main }), "Color", "off")
uiScale = create("UIScale", { Scale = 1, Parent = Main })

TitleBar = create("Frame", {
	Name = "TitleBar",
	Size = UDim2.new(1, 0, 0, 40),
	BorderSizePixel = 0,
	Parent = Main,
})
reg(TitleBar, "BackgroundColor3", "title")
regTransp(TitleBar)

reg(create("TextLabel", {
	BackgroundTransparency = 1,
	Position = UDim2.fromOffset(12, 0),
	Size = UDim2.new(1, -100, 1, 0),
	Text = "Menu de Utilidades",
	TextXAlignment = Enum.TextXAlignment.Left,
	Font = Enum.Font.GothamBold,
	TextSize = 15,
	Parent = TitleBar,
}), "TextColor3", "text")

MinBtn = create("TextButton", {
	AnchorPoint = Vector2.new(1, 0.5),
	Position = UDim2.new(1, -46, 0.5, 0),
	Size = UDim2.fromOffset(34, 30),
	Text = "–",
	Font = Enum.Font.GothamBold,
	TextSize = 18,
	AutoButtonColor = true,
	BorderSizePixel = 0,
	Parent = TitleBar,
})
reg(MinBtn, "BackgroundColor3", "panel")
reg(MinBtn, "TextColor3", "text")
corner(MinBtn, 6)

local CloseBtn = create("TextButton", {
	AnchorPoint = Vector2.new(1, 0.5),
	Position = UDim2.new(1, -6, 0.5, 0),
	Size = UDim2.fromOffset(34, 30),
	Text = "✕",
	Font = Enum.Font.GothamBold,
	TextSize = 15,
	AutoButtonColor = true,
	BorderSizePixel = 0,
	Parent = TitleBar,
})
reg(CloseBtn, "BackgroundColor3", "panel")
reg(CloseBtn, "TextColor3", "text")
corner(CloseBtn, 6)

Body = create("ScrollingFrame", {
	Name = "Body",
	Position = UDim2.fromOffset(0, 76),
	Size = UDim2.new(1, 0, 1, -76),
	BackgroundTransparency = 1,
	BorderSizePixel = 0,
	ScrollBarThickness = 5,
	CanvasSize = UDim2.new(0, 0, 0, 0),
	AutomaticCanvasSize = Enum.AutomaticSize.Y,
	ScrollingDirection = Enum.ScrollingDirection.Y,
	Parent = Main,
})
reg(Body, "ScrollBarImageColor3", "accent")
create("UIListLayout", { Padding = UDim.new(0, 8), SortOrder = Enum.SortOrder.LayoutOrder, Parent = Body })
create("UIPadding", {
	PaddingTop = UDim.new(0, 8),
	PaddingBottom = UDim.new(0, 12),
	PaddingLeft = UDim.new(0, 8),
	PaddingRight = UDim.new(0, 10),
	Parent = Body,
})

local TabBar = create("Frame", {
	Name = "TabBar",
	Position = UDim2.fromOffset(8, 44),
	Size = UDim2.new(1, -16, 0, 28),
	BackgroundTransparency = 1,
	Parent = Main,
})
create("UIListLayout", {
	FillDirection = Enum.FillDirection.Horizontal,
	HorizontalAlignment = Enum.HorizontalAlignment.Center,
	Padding = UDim.new(0, 5),
	SortOrder = Enum.SortOrder.LayoutOrder,
	Parent = TabBar,
})

-- Botão flutuante para reabrir o menu (aparece quando a interface está OFF)
OpenBtn = create("TextButton", {
	Name = "OpenBtn",
	Position = UDim2.fromOffset(12, 12),
	Size = UDim2.fromOffset(64, 36),
	Text = "MENU",
	Font = Enum.Font.GothamBold,
	TextSize = 13,
	TextColor3 = Color3.new(1, 1, 1),
	AutoButtonColor = true,
	BorderSizePixel = 0,
	Visible = false,
	Parent = gui,
})
reg(OpenBtn, "BackgroundColor3", "accent")
corner(OpenBtn, 8)

-- HUD (FPS / Ping / Coordenadas / Jogadores)
HudFrame = create("Frame", {
	Name = "Hud",
	AnchorPoint = Vector2.new(1, 0),
	Position = UDim2.new(1, -8, 0, 8),
	Size = UDim2.fromOffset(0, 0),
	AutomaticSize = Enum.AutomaticSize.XY,
	BackgroundTransparency = 0.25,
	BorderSizePixel = 0,
	Visible = false,
	Parent = gui,
})
reg(HudFrame, "BackgroundColor3", "bg")
corner(HudFrame, 8)
create("UIPadding", {
	PaddingTop = UDim.new(0, 6),
	PaddingBottom = UDim.new(0, 6),
	PaddingLeft = UDim.new(0, 8),
	PaddingRight = UDim.new(0, 8),
	Parent = HudFrame,
})
create("UIListLayout", {
	Padding = UDim.new(0, 2),
	SortOrder = Enum.SortOrder.LayoutOrder,
	HorizontalAlignment = Enum.HorizontalAlignment.Right,
	Parent = HudFrame,
})
for i, key in ipairs({ "ShowFPS", "ShowPing", "ShowCoords", "PlayerCount" }) do
	local l = create("TextLabel", {
		AutomaticSize = Enum.AutomaticSize.XY,
		Size = UDim2.fromOffset(0, 0),
		BackgroundTransparency = 1,
		Font = Enum.Font.GothamMedium,
		TextSize = 14,
		TextXAlignment = Enum.TextXAlignment.Right,
		Text = "...",
		Visible = false,
		LayoutOrder = i,
		Parent = HudFrame,
	})
	reg(l, "TextColor3", "text")
	hudLabels[key] = l
end

-- Container das notificações
notifHolder = create("Frame", {
	Name = "Notifications",
	AnchorPoint = Vector2.new(0.5, 0),
	Position = UDim2.new(0.5, 0, 0, 6),
	Size = UDim2.fromOffset(260, 0),
	AutomaticSize = Enum.AutomaticSize.Y,
	BackgroundTransparency = 1,
	ZIndex = 50,
	Parent = gui,
})
create("UIListLayout", {
	Padding = UDim.new(0, 4),
	SortOrder = Enum.SortOrder.LayoutOrder,
	HorizontalAlignment = Enum.HorizontalAlignment.Center,
	Parent = notifHolder,
})

--==================================================================
-- Layout (tamanho/escala responsivos)
--==================================================================
local function updateLayout(animate)
	local cam = workspace.CurrentCamera
	local vp = cam and cam.ViewportSize or Vector2.new(800, 600)
	local sc = S.Scale
	local w = math.clamp((vp.X - 24) / sc, 220, 340)
	local h = math.clamp((vp.Y - 90) / sc, 140, 480)
	local size = UDim2.fromOffset(w, S.Minimized and 40 or h)
	uiScale.Scale = sc
	Body.Position = UDim2.fromOffset(0, 76)
	Body.Size = UDim2.new(1, 0, 1, -76)
	if animate then
		tween(Main, { Size = size }, 0.18)
	else
		Main.Size = size
	end
	if teamsApi.layout then
		teamsApi.layout()
	end
end

--==================================================================
-- Arrastar (mouse e toque)
--==================================================================
local function bindDrag(target, onBegin, onMove, onEnd)
	local active, current = false, nil
	connect(target.InputBegan, function(input)
		local t = input.UserInputType
		if t == Enum.UserInputType.MouseButton1 or t == Enum.UserInputType.Touch then
			if onBegin and onBegin(input) == false then
				return
			end
			active, current = true, input
			onMove(input)
		end
	end)
	connect(UserInputService.InputChanged, function(input)
		if not active then
			return
		end
		if input == current or input.UserInputType == Enum.UserInputType.MouseMovement then
			onMove(input)
		end
	end)
	connect(UserInputService.InputEnded, function(input)
		if not active then
			return
		end
		local t = input.UserInputType
		if input == current or (current.UserInputType == Enum.UserInputType.MouseButton1 and t == Enum.UserInputType.MouseButton1) then
			active, current = false, nil
			if onEnd then
				onEnd()
			end
		end
	end)
end

do
	local startPos, startMouse
	bindDrag(TitleBar, function(input)
		if not S.Draggable then
			return false
		end
		startPos = Main.Position
		startMouse = Vector2.new(input.Position.X, input.Position.Y)
		return true
	end, function(input)
		if not startPos then
			return
		end
		local cam = workspace.CurrentCamera
		local vp = cam and cam.ViewportSize or Vector2.new(800, 600)
		local dx = input.Position.X - startMouse.X
		local dy = input.Position.Y - startMouse.Y
		local x = math.clamp(startPos.X.Offset + dx, 0, math.max(vp.X - 60, 0))
		local y = math.clamp(startPos.Y.Offset + dy, 0, math.max(vp.Y - 100, 0))
		Main.Position = UDim2.fromOffset(x, y)
	end)
end

--==================================================================
-- Construtores de controles: Categoria, Toggle, Slider, Button, Dropdown
--==================================================================
local function addCategory(name)
	local wrap = create("Frame", {
		Size = UDim2.new(1, 0, 0, 0),
		AutomaticSize = Enum.AutomaticSize.Y,
		BackgroundTransparency = 1,
		LayoutOrder = nextOrder(),
		Parent = Body,
	})
	create("UIListLayout", { Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder, Parent = wrap })

	local header = create("TextButton", {
		Size = UDim2.new(1, 0, 0, 32),
		Text = "▼  " .. name,
		Font = Enum.Font.GothamBold,
		TextSize = 13,
		TextXAlignment = Enum.TextXAlignment.Left,
		AutoButtonColor = false,
		BorderSizePixel = 0,
		LayoutOrder = 0,
		Parent = wrap,
	})
	reg(header, "BackgroundColor3", "title")
	reg(header, "TextColor3", "accent")
	corner(header, 8)
	create("UIPadding", { PaddingLeft = UDim.new(0, 12), Parent = header })

	local items = create("Frame", {
		Size = UDim2.new(1, 0, 0, 0),
		AutomaticSize = Enum.AutomaticSize.Y,
		BackgroundTransparency = 1,
		LayoutOrder = 1,
		Parent = wrap,
	})
	create("UIListLayout", { Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder, Parent = items })

	connect(header.Activated, function()
		playClick()
		items.Visible = not items.Visible
		header.Text = (items.Visible and "▼  " or "▶  ") .. name
	end)
	categoryRegistry[name] = wrap
	return items
end

local function makeRow(parent, height)
	local row = create("Frame", {
		Size = UDim2.new(1, 0, 0, height),
		BorderSizePixel = 0,
		LayoutOrder = nextOrder(),
		Parent = parent,
	})
	reg(row, "BackgroundColor3", "panel")
	regTransp(row)
	corner(row, 8)
	return row
end

local function addToggle(parent, label, key)
	toggleNames[key] = label
	local row = makeRow(parent, 44)

	reg(create("TextLabel", {
		BackgroundTransparency = 1,
		Position = UDim2.fromOffset(12, 0),
		Size = UDim2.new(1, -74, 1, 0),
		Text = label,
		TextWrapped = true,
		TextXAlignment = Enum.TextXAlignment.Left,
		Font = Enum.Font.GothamMedium,
		TextSize = 14,
		Parent = row,
	}), "TextColor3", "text")

	local pill = create("Frame", {
		AnchorPoint = Vector2.new(1, 0.5),
		Position = UDim2.new(1, -10, 0.5, 0),
		Size = UDim2.fromOffset(46, 24),
		BorderSizePixel = 0,
		Parent = row,
	})
	corner(pill, 12)
	local knob = create("Frame", {
		AnchorPoint = Vector2.new(0, 0.5),
		Position = UDim2.new(0, 2, 0.5, 0),
		Size = UDim2.fromOffset(20, 20),
		BackgroundColor3 = Color3.new(1, 1, 1),
		BorderSizePixel = 0,
		Parent = pill,
	})
	corner(knob, 10)

	local btn = create("TextButton", {
		Size = UDim2.fromScale(1, 1),
		BackgroundTransparency = 1,
		Text = "",
		AutoButtonColor = false,
		ZIndex = 5,
		Parent = row,
	})

	refreshers[key] = function(instant)
		local on = S[key]
		local pos = on and UDim2.new(1, -22, 0.5, 0) or UDim2.new(0, 2, 0.5, 0)
		local col = on and T().accent or T().off
		if instant then
			pill.BackgroundColor3 = col
			knob.Position = pos
		else
			tween(pill, { BackgroundColor3 = col }, 0.15)
			tween(knob, { Position = pos }, 0.15)
		end
	end
	refreshers[key](true)

	connect(btn.Activated, function()
		playClick()
		setSetting(key, not S[key])
	end)
end

local function addSlider(parent, label, key, min, max, step, fmt)
	local row = makeRow(parent, 62)
	fmt = fmt or function(v)
		if step >= 1 then
			return tostring(math.floor(v + 0.5))
		end
		return string.format("%.1f", v)
	end

	local lbl = create("TextLabel", {
		BackgroundTransparency = 1,
		Position = UDim2.fromOffset(12, 4),
		Size = UDim2.new(1, -90, 0, 22),
		Text = label,
		TextXAlignment = Enum.TextXAlignment.Left,
		TextTruncate = Enum.TextTruncate.AtEnd,
		Font = Enum.Font.GothamMedium,
		TextSize = 14,
		Parent = row,
	})
	reg(lbl, "TextColor3", "text")

	local val = create("TextLabel", {
		AnchorPoint = Vector2.new(1, 0),
		BackgroundTransparency = 1,
		Position = UDim2.new(1, -12, 0, 4),
		Size = UDim2.fromOffset(70, 22),
		TextXAlignment = Enum.TextXAlignment.Right,
		Font = Enum.Font.GothamBold,
		TextSize = 14,
		Text = "",
		Parent = row,
	})
	reg(val, "TextColor3", "accent")

	local track = create("Frame", {
		BackgroundTransparency = 1,
		Position = UDim2.fromOffset(14, 32),
		Size = UDim2.new(1, -28, 0, 24),
		Parent = row,
	})
	local bar = create("Frame", {
		AnchorPoint = Vector2.new(0, 0.5),
		Position = UDim2.new(0, 0, 0.5, 0),
		Size = UDim2.new(1, 0, 0, 6),
		BorderSizePixel = 0,
		Parent = track,
	})
	reg(bar, "BackgroundColor3", "off")
	corner(bar, 3)
	local fill = create("Frame", { Size = UDim2.new(0, 0, 1, 0), BorderSizePixel = 0, Parent = bar })
	reg(fill, "BackgroundColor3", "accent")
	corner(fill, 3)
	local knob = create("Frame", {
		AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.new(0, 0, 0.5, 0),
		Size = UDim2.fromOffset(20, 20),
		BackgroundColor3 = Color3.new(1, 1, 1),
		BorderSizePixel = 0,
		Parent = bar,
	})
	corner(knob, 10)

	refreshers[key] = function()
		local v = math.clamp(S[key], min, max)
		local r = (v - min) / (max - min)
		fill.Size = UDim2.new(r, 0, 1, 0)
		knob.Position = UDim2.new(r, 0, 0.5, 0)
		val.Text = fmt(S[key])
	end
	refreshers[key]()

	bindDrag(track, function()
		Body.ScrollingEnabled = false
		return true
	end, function(input)
		local rel = math.clamp((input.Position.X - track.AbsolutePosition.X) / math.max(track.AbsoluteSize.X, 1), 0, 1)
		local v = min + rel * (max - min)
		v = min + math.floor((v - min) / step + 0.5) * step
		v = math.clamp(math.floor(v * 1000 + 0.5) / 1000, min, max)
		if v ~= S[key] then
			setSetting(key, v, true)
		end
	end, function()
		Body.ScrollingEnabled = true
	end)
end

local function addButton(parent, label, callback)
	local btn = create("TextButton", {
		Size = UDim2.new(1, 0, 0, 42),
		Text = label,
		Font = Enum.Font.GothamBold,
		TextSize = 14,
		TextColor3 = Color3.new(1, 1, 1),
		AutoButtonColor = true,
		BorderSizePixel = 0,
		LayoutOrder = nextOrder(),
		Parent = parent,
	})
	reg(btn, "BackgroundColor3", "accent")
	corner(btn, 8)
	connect(btn.Activated, function()
		playClick()
		callback()
	end)
end

local function addDropdown(parent, label, key, options)
	local holder = makeRow(parent, 44)
	holder.ClipsDescendants = true

	reg(create("TextLabel", {
		BackgroundTransparency = 1,
		Position = UDim2.fromOffset(12, 0),
		Size = UDim2.new(1, -150, 0, 44),
		Text = label,
		TextXAlignment = Enum.TextXAlignment.Left,
		TextWrapped = true,
		Font = Enum.Font.GothamMedium,
		TextSize = 14,
		Parent = holder,
	}), "TextColor3", "text")

	local head = create("TextButton", {
		AnchorPoint = Vector2.new(1, 0),
		Position = UDim2.new(1, -8, 0, 7),
		Size = UDim2.fromOffset(130, 30),
		Font = Enum.Font.GothamBold,
		TextSize = 13,
		AutoButtonColor = false,
		BorderSizePixel = 0,
		Text = "",
		Parent = holder,
	})
	reg(head, "BackgroundColor3", "off")
	reg(head, "TextColor3", "text")
	corner(head, 6)

	local list = create("Frame", {
		Position = UDim2.fromOffset(8, 48),
		Size = UDim2.new(1, -16, 0, #options * 34),
		BackgroundTransparency = 1,
		Parent = holder,
	})
	create("UIListLayout", { Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder, Parent = list })

	local optBtns = {}
	local open = false
	local function setOpen(o)
		open = o
		tween(holder, { Size = UDim2.new(1, 0, 0, 44 + (o and (#options * 34 + 8) or 0)) }, 0.15)
	end

	for i, opt in ipairs(options) do
		local b = create("TextButton", {
			Size = UDim2.new(1, 0, 0, 30),
			Text = opt,
			Font = Enum.Font.GothamMedium,
			TextSize = 14,
			AutoButtonColor = false,
			BorderSizePixel = 0,
			LayoutOrder = i,
			Parent = list,
		})
		corner(b, 6)
		optBtns[opt] = b
		connect(b.Activated, function()
			playClick()
			setOpen(false)
			setSetting(key, opt)
		end)
	end

	connect(head.Activated, function()
		playClick()
		setOpen(not open)
	end)

	refreshers[key] = function()
		head.Text = tostring(S[key]) .. "  ▼"
		local th = T()
		for opt, b in pairs(optBtns) do
			local sel = (opt == S[key])
			b.BackgroundColor3 = sel and th.accent or th.off
			b.TextColor3 = sel and Color3.new(1, 1, 1) or th.text
		end
	end
	refreshers[key]()
end

--==================================================================
-- Lógica: personagem
--==================================================================
local lastAliveCF = nil
local pendingCF = nil
local shiftHum = nil

local function enforceCharacter(hum)
	if S.WalkSpeed ~= D.WalkSpeed and hum.WalkSpeed ~= S.WalkSpeed then
		hum.WalkSpeed = S.WalkSpeed
	end
	if S.JumpPower ~= D.JumpPower then
		if not hum.UseJumpPower then
			hum.UseJumpPower = true
		end
		if hum.JumpPower ~= S.JumpPower then
			hum.JumpPower = S.JumpPower
		end
	end
	if (S.ShiftLock or not S.AutoRotate) and hum.AutoRotate then
		hum.AutoRotate = false
	end
end

local function resetCharacter()
	local hum = getHum()
	local char = LocalPlayer.Character
	if hum then
		hum.Health = 0
	end
	if char then
		pcall(function()
			char:BreakJoints()
		end)
	end
	notify("Personagem resetado", true)
end

local function onCharacter(char)
	local hum = char:WaitForChild("Humanoid", 10)
	if not hum then
		return
	end
	connect(hum.Died, function()
		if S.RespawnFast and lastAliveCF then
			pendingCF = lastAliveCF
		end
	end)
	if pendingCF and S.RespawnFast then
		local cf = pendingCF
		pendingCF = nil
		local root = char:WaitForChild("HumanoidRootPart", 10)
		if root then
			task.wait(0.2)
			if S.RespawnFast and char.Parent then
				root.CFrame = cf + Vector3.new(0, 3, 0)
			end
		end
	else
		pendingCF = nil
	end
end

connect(LocalPlayer.CharacterAdded, function(char)
	task.spawn(onCharacter, char)
	FlyTargetCFrame = nil
	FrozenCFrame = nil
	task.spawn(function()
		local root = char:WaitForChild("HumanoidRootPart", 5)
		if root and FlyEnabled then root.Anchored = true end
	end)
end)
if LocalPlayer.Character then
	task.spawn(onCharacter, LocalPlayer.Character)
end

--==================================================================
-- Lógica: câmera
--==================================================================
local function applyZoom()
	local mn = S.MinZoom
	local mx = math.max(S.MaxZoom, mn)
	pcall(function()
		if mx >= LocalPlayer.CameraMinZoomDistance then
			LocalPlayer.CameraMaxZoomDistance = mx
			LocalPlayer.CameraMinZoomDistance = mn
		else
			LocalPlayer.CameraMinZoomDistance = mn
			LocalPlayer.CameraMaxZoomDistance = mx
		end
	end)
end

RunService:BindToRenderStep("MenuUtil30_Render", Enum.RenderPriority.Camera.Value + 1, function()
	local cam = workspace.CurrentCamera
	if cam and S.FOV ~= D.FOV and cam.FieldOfView ~= S.FOV then
		cam.FieldOfView = S.FOV
	end

	if S.ShiftLock then
		local hum, root = getHum(), getRoot()
		if cam and hum and root and hum.Health > 0 then
			if shiftHum ~= hum then
				hum.CameraOffset = Vector3.new(1.75, 0, 0)
				shiftHum = hum
			end
			if not hum.Sit then
				local look = cam.CFrame.LookVector
				local flat = Vector3.new(look.X, 0, look.Z)
				if flat.Magnitude > 0.01 then
					root.CFrame = CFrame.new(root.Position, root.Position + flat.Unit)
				end
			end
		end
	end
end)

--==================================================================
-- Lógica: jogadores (nomes, distância, destaque)
--==================================================================
local esp = {}

local function clearESP(plr)
	local e = esp[plr]
	if e then
		if e.hl then
			e.hl:Destroy()
		end
		if e.bb then
			e.bb:Destroy()
		end
		esp[plr] = nil
	end
end

local function updateESP(plr, myRoot)
	local char = plr.Character
	local head = char and (char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart"))
	if not char or not head then
		clearESP(plr)
		return
	end
	local e = esp[plr]
	if not e then
		e = {}
		esp[plr] = e
	end
	local th = T()

	local function teamESPColor(target)
		local myTeam = LocalPlayer.Team
		local targetTeam = target.Team
		if myTeam and targetTeam then
			if targetTeam == myTeam then
				return myTeam.TeamColor.Color
			end
			return targetTeam.TeamColor.Color
		end
		if targetTeam then
			return targetTeam.TeamColor.Color
		end
		if not target.Neutral and target.TeamColor then
			return target.TeamColor.Color
		end
		return th.sub
	end
	local espColor = teamESPColor(plr)

	-- Destaque
	if S.Highlight then
		if not e.hl or e.hl.Parent ~= char then
			if e.hl then
				e.hl:Destroy()
			end
			e.hl = create("Highlight", {
				Name = "MU_Highlight",
				Adornee = char,
				FillColor = espColor,
				FillTransparency = 0.65,
				OutlineColor = Color3.new(1, 1, 1),
				OutlineTransparency = 0,
				DepthMode = Enum.HighlightDepthMode.AlwaysOnTop,
				Parent = char,
			})
		end
		e.hl.FillColor = espColor
	elseif e.hl then
		e.hl:Destroy()
		e.hl = nil
	end

	-- Nome / distância
	if S.ShowNames or S.ShowDistance then
		if not e.bb or e.bb.Parent ~= head then
			if e.bb then
				e.bb:Destroy()
			end
			local bb = create("BillboardGui", {
				Name = "MU_Info",
				Adornee = head,
				Size = UDim2.fromOffset(200, 44),
				StudsOffset = Vector3.new(0, 2.6, 0),
				AlwaysOnTop = true,
				ResetOnSpawn = false,
				Parent = head,
			})
			e.label = create("TextLabel", {
				BackgroundTransparency = 1,
				Size = UDim2.fromScale(1, 1),
				Font = Enum.Font.GothamBold,
				TextSize = 14,
				TextColor3 = Color3.new(1, 1, 1),
				TextStrokeTransparency = 0.4,
				Text = "",
				Parent = bb,
			})
			e.bb = bb
		end
		local lines = {}
		if S.ShowNames then
			table.insert(lines, plr.DisplayName ~= "" and plr.DisplayName or plr.Name)
		end
		if S.ShowDistance then
			local r = char:FindFirstChild("HumanoidRootPart")
			if r and myRoot then
				table.insert(lines, string.format("%d studs", math.floor((r.Position - myRoot.Position).Magnitude + 0.5)))
			else
				table.insert(lines, "-- studs")
			end
		end
		local txt = table.concat(lines, "\n")
		e.label.TextColor3 = espColor
		if e.label.Text ~= txt then
			e.label.Text = txt
		end
	elseif e.bb then
		e.bb:Destroy()
		e.bb = nil
		e.label = nil
	end
end

local function refreshESP()
	if not (S.Highlight or S.ShowNames or S.ShowDistance) then
		for plr in pairs(esp) do
			clearESP(plr)
		end
		return
	end
	local myRoot = getRoot()
	for _, plr in ipairs(Players:GetPlayers()) do
		if plr ~= LocalPlayer then
			updateESP(plr, myRoot)
		end
	end
end

connect(Players.PlayerRemoving, clearESP)

--==================================================================
-- Lógica: HUD (FPS, Ping, Coordenadas, Jogadores)
--==================================================================
local lastFps = 60

local function getPing()
	local ok, v = pcall(function()
		return Stats.Network.ServerStatsItem["Data Ping"]:GetValue()
	end)
	if ok and typeof(v) == "number" then
		return v
	end
	local ok2, v2 = pcall(function()
		return LocalPlayer:GetNetworkPing() * 1000
	end)
	if ok2 and typeof(v2) == "number" then
		return v2
	end
	return nil
end

local function updateHud()
	if S.ShowFPS then
		hudLabels.ShowFPS.Text = string.format("FPS: %d", math.floor(lastFps + 0.5))
	end
	if S.ShowPing then
		local p = getPing()
		hudLabels.ShowPing.Text = p and string.format("Ping: %d ms", math.floor(p + 0.5)) or "Ping: --"
	end
	if S.ShowCoords then
		local r = getRoot()
		if r then
			local pos = r.Position
			hudLabels.ShowCoords.Text = string.format("XYZ: %.1f, %.1f, %.1f", pos.X, pos.Y, pos.Z)
		else
			hudLabels.ShowCoords.Text = "XYZ: --"
		end
	end
	if S.PlayerCount then
		hudLabels.PlayerCount.Text = string.format("Jogadores: %d/%d", #Players:GetPlayers(), Players.MaxPlayers)
	end
end

local function applyHudVisibility()
	for key, lbl in pairs(hudLabels) do
		lbl.Visible = S[key] == true
	end
	HudFrame.Visible = S.ShowFPS or S.ShowPing or S.ShowCoords or S.PlayerCount
	updateHud()
end

--==================================================================
-- Anti-AFK
--==================================================================
connect(LocalPlayer.Idled, function()
	if S.AntiAFK then
		pcall(function()
			VirtualUser:CaptureController()
			VirtualUser:ClickButton2(Vector2.new(0, 0))
		end)
	end
end)

--==================================================================
-- Salvar / Restaurar configurações
--==================================================================
saveConfig = function()
	local copy = table.clone(S)
	copy.InterfaceOn = true
	copy.SavedPosition = nil
	local ok, data = pcall(function()
		return HttpService:JSONEncode(copy)
	end)
	if not ok then
		notify("Erro ao salvar configurações", true)
		return
	end
	if HAS_FS then
		local ok2 = pcall(writefile, FILE, data)
		notify(ok2 and "Configurações salvas em arquivo" or "Falha ao gravar o arquivo", true)
	else
		shared.MenuUtil30_Saved = data
		notify("Salvo nesta sessão (sem writefile)", true)
	end
end

local function restoreDefaults()
	for k, v in pairs(D) do
		setSetting(k, v, true)
	end
	refreshAll(true)
	notify("Configurações restauradas", true)
end

--==================================================================
-- MOVIMENTO / FLY + POSIÇÃO SALVA
--==================================================================
local FlyEnabled = S.FlyEnabled == true
local FlyFrozen = S.FlyFrozen == true
local FlySpeed = tonumber(S.FlySpeed) or 100
local FrozenCFrame = nil
local FlyTargetCFrame = nil

local function usandoJoystick()
	return UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled
end

local function savePosition()
	local root = getRoot()
	if not root then
		notify("Posição não salva: personagem não encontrado", true)
		return
	end
	S.SavedPosition = root.CFrame
	notify("Posição salva!", true)
end

local function goToSavedPosition()
	if typeof(S.SavedPosition) ~= "CFrame" then
		notify("Nenhuma posição salva.", true)
		return
	end
	if not getRoot() then
		notify("Personagem não encontrado.", true)
		return
	end
	FlyTargetCFrame = S.SavedPosition
	if not FlyEnabled then
		FlyEnabled = true
		S.FlyEnabled = true
	end
	FlyFrozen = false
	S.FlyFrozen = false
	FrozenCFrame = nil
	notify("Voando até a posição salva...", true)
end

local function setFlyEnabled(value)
	FlyEnabled = value == true
	S.FlyEnabled = FlyEnabled
	if not FlyEnabled then
		FlyFrozen = false
		S.FlyFrozen = false
		FrozenCFrame = nil
		FlyTargetCFrame = nil
		local root = getRoot()
		if root then pcall(function() root.Anchored = false end) end
	end
end

local function toggleFly()
	setFlyEnabled(not FlyEnabled)
	notify("Fly: " .. (FlyEnabled and "ON" or "OFF"), true)
end

local function setFlyFrozen(value)
	if not FlyEnabled then return end
	FlyFrozen = value == true
	S.FlyFrozen = FlyFrozen
	if FlyFrozen then
		local root = getRoot()
		FrozenCFrame = root and root.CFrame or nil
	else
		FrozenCFrame = nil
	end
end

local function setFlySpeed(value)
	value = math.clamp(tonumber(value) or 100, 10, 500)
	FlySpeed = value
	S.FlySpeed = value
end

local function updateFly(dt)
	if not FlyEnabled then return end
	local character = LocalPlayer.Character
	if not character then return end
	local root = character:FindFirstChild("HumanoidRootPart")
	local humanoid = character:FindFirstChildOfClass("Humanoid")
	if not root or not humanoid then return end

	root.Anchored = true

	if FlyTargetCFrame then
		local delta = FlyTargetCFrame.Position - root.Position
		local distance = delta.Magnitude
		if distance <= math.max(1.5, FlySpeed * dt) then
			root.CFrame = FlyTargetCFrame
			FlyTargetCFrame = nil
			notify("Chegou à posição salva.", true)
			return
		end
		if distance > 0 then
			root.CFrame = CFrame.lookAt(root.Position, FlyTargetCFrame.Position) + delta.Unit * FlySpeed * dt
		end
		return
	end

	if FlyFrozen then
		if FrozenCFrame then root.CFrame = FrozenCFrame end
		return
	end

	local camera = workspace.CurrentCamera
	if not camera then return end
	local lookVector = camera.CFrame.LookVector
	local rightVector = camera.CFrame.RightVector
	local direction = Vector3.zero

	if usandoJoystick() then
		local moveDir = humanoid.MoveDirection
		local flatLook = Vector3.new(lookVector.X, 0, lookVector.Z)
		local flatRight = Vector3.new(rightVector.X, 0, rightVector.Z)
		if flatLook.Magnitude > 0 then flatLook = flatLook.Unit end
		if flatRight.Magnitude > 0 then flatRight = flatRight.Unit end
		local forwardAmount = moveDir:Dot(flatLook)
		local strafeAmount = moveDir:Dot(flatRight)
		direction = (lookVector * forwardAmount) + (rightVector * strafeAmount)
	else
		if UserInputService:IsKeyDown(Enum.KeyCode.W) then direction += lookVector end
		if UserInputService:IsKeyDown(Enum.KeyCode.S) then direction -= lookVector end
		if UserInputService:IsKeyDown(Enum.KeyCode.A) then direction -= rightVector end
		if UserInputService:IsKeyDown(Enum.KeyCode.D) then direction += rightVector end
	end

	if direction.Magnitude > 0 then
		direction = direction.Unit
		root.CFrame = root.CFrame + direction * FlySpeed * dt
	end
end

local function setESPAll(value)
	local on = value == true
	S.Quick_ESP = on
	setSetting("Highlight", on, true)
	setSetting("ShowNames", on, true)
	setSetting("ShowDistance", on, true)
end

--==================================================================
-- Montagem das categorias e das opções
--==================================================================
do
	-- PERSONAGEM (5)
	local c = addCategory("PERSONAGEM")
	addSlider(c, "WalkSpeed", "WalkSpeed", 0, 200, 1)
	addSlider(c, "JumpPower", "JumpPower", 0, 300, 1)
	addToggle(c, "Rotação automática do personagem", "AutoRotate")
	addToggle(c, "Respawn rápido (volta ao local da morte)", "RespawnFast")
	addButton(c, "Resetar personagem", resetCharacter)

	-- CÂMERA (6)
	c = addCategory("CÂMERA")
	addSlider(c, "FOV da câmera", "FOV", 30, 120, 1)
	addSlider(c, "Zoom máximo da câmera", "MaxZoom", 10, 1000, 1)
	addSlider(c, "Zoom mínimo da câmera", "MinZoom", 0.5, 30, 0.5)
	addSlider(c, "Sensib. da câmera (controle)", "CamSens", 0.1, 5, 0.1)
	addSlider(c, "Sensibilidade do mouse", "MouseSens", 0.1, 5, 0.1)
	addToggle(c, "Shift Lock (personagem segue a câmera)", "ShiftLock")

	-- JOGADORES / ESP
	c = addCategory("JOGADORES / ESP")
	addToggle(c, "Mostrar nome dos jogadores", "ShowNames")
	addToggle(c, "Mostrar distância dos jogadores", "ShowDistance")
	addToggle(c, "Destacar jogadores por time", "Highlight")
	addToggle(c, "Contador de jogadores", "PlayerCount")
	addToggle(c, "ESP inteligente por time", "Quick_ESP")
	addButton(c, "Ver times", function()
		setSetting("DetectTeams", true)
	end)

	-- Função antiga preservada.
	c = addCategory("TIMES")
	addToggle(c, "Detectar Times", "DetectTeams")

	-- MOVIMENTO
	c = addCategory("MOVIMENTO")
	addToggle(c, "Fly", "FlyEnabled")
	addToggle(c, "Noclip", "Noclip")
	addToggle(c, "Congelar Fly", "FlyFrozen")
	addSlider(c, "Velocidade do Fly", "FlySpeed", 10, 500, 1)
	addButton(c, "Salvar posição", savePosition)
	addButton(c, "Ir para posição salva", goToSavedPosition)

	-- DESEMPENHO (3)
	c = addCategory("DESEMPENHO")
	addToggle(c, "Mostrar FPS", "ShowFPS")
	addToggle(c, "Mostrar Ping", "ShowPing")
	addToggle(c, "Mostrar coordenadas", "ShowCoords")

	-- UTILIDADES (1)
	c = addCategory("UTILIDADES")
	addToggle(c, "Anti-AFK", "AntiAFK")

	-- INTERFACE (9)
	c = addCategory("INTERFACE")
	addToggle(c, "Interface ON/OFF", "InterfaceOn")
	addToggle(c, "Minimizar menu", "Minimized")
	addToggle(c, "Arrastar menu", "Draggable")
	addSlider(c, "Transparência da interface", "Transparency", 0, 0.7, 0.05, function(v)
		return math.floor(v * 100 + 0.5) .. "%"
	end)
	addSlider(c, "Tamanho da interface", "Scale", 0.7, 1.3, 0.05, function(v)
		return string.format("%.2fx", v)
	end)
	addDropdown(c, "Tema da interface", "Theme", THEME_ORDER)
	addToggle(c, "Animações da interface", "Animations")
	addToggle(c, "Som da interface", "Sound")
	addToggle(c, "Notificações", "Notifications")
	addButton(c, "Editar Menu Rápido", function()
		quickEditor.Visible = not quickEditor.Visible
	end)

	-- CONFIGURAÇÕES (2)
	c = addCategory("CONFIGURAÇÕES")
	addButton(c, "Restaurar configurações", restoreDefaults)
	addButton(c, "Salvar configurações", saveConfig)
end

--==================================================================
-- DETECTAR TIMES (módulo novo)
-- Analisa Teams, Players e Workspace sem nomes de equipes fixos.
--==================================================================
do
	-- Nomes de CHAVES (não de equipes) que jogos costumam usar para guardar o time
	local TEAM_ATTRS = { "Team", "TeamName", "team", "teamName", "Time", "Equipe" }
	local TEAM_VALUE_NAMES = { team = true, teamname = true, time = true, equipe = true }
	local COLOR_ATTRS = { "TeamColor", "Color", "BrickColor" }
	local COLOR_TOLERANCE = 0.06

	local destroyed = false
	local wsCache = { named = {}, spawns = {}, colored = {} }
	local lastSig = nil
	local wsConns = {}

	local function hex(c)
		return string.format("#%02X%02X%02X", math.floor(c.R * 255 + 0.5), math.floor(c.G * 255 + 0.5), math.floor(c.B * 255 + 0.5))
	end

	local function colorDist(a, b)
		return math.abs(a.R - b.R) + math.abs(a.G - b.G) + math.abs(a.B - b.B)
	end

	local function attrName(inst)
		for _, k in ipairs(TEAM_ATTRS) do
			local v = inst:GetAttribute(k)
			if typeof(v) == "string" and v ~= "" then
				return v
			end
		end
		return nil
	end

	local function valueTeamName(cont)
		if not cont then
			return nil
		end
		for _, ch in ipairs(cont:GetChildren()) do
			if TEAM_VALUE_NAMES[ch.Name:lower()] then
				if ch:IsA("StringValue") and ch.Value ~= "" then
					return ch.Value
				elseif ch:IsA("ObjectValue") and ch.Value then
					return ch.Value.Name
				end
			end
		end
		return nil
	end

	-- Retorna: nome do time, fonte, cor (se conhecida)
	local function playerTeamInfo(plr)
		local tm = plr.Team
		if tm then
			return tm.Name, "Teams", tm.TeamColor.Color
		end
		local n = attrName(plr)
		if n then
			return n, "Atributo", nil
		end
		local n2 = valueTeamName(plr) or valueTeamName(plr:FindFirstChild("leaderstats"))
		if n2 then
			return n2, "Valor do jogador", nil
		end
		local char = plr.Character
		if char then
			local n3 = attrName(char)
			if n3 then
				return n3, "Atributo", nil
			end
		end
		return nil
	end

	-- Cor "principal" de uma instância (quando existir)
	local function instColor(inst)
		if inst:IsA("BasePart") then
			return inst.Color
		end
		if inst:IsA("Color3Value") then
			return inst.Value
		end
		if inst:IsA("BrickColorValue") then
			return inst.Value.Color
		end
		for _, k in ipairs(COLOR_ATTRS) do
			local v = inst:GetAttribute(k)
			if typeof(v) == "Color3" then
				return v
			elseif typeof(v) == "BrickColor" then
				return v.Color
			end
		end
		if inst:IsA("Model") or inst:IsA("Folder") or inst:IsA("Configuration") then
			for _, ch in ipairs(inst:GetChildren()) do
				if ch:IsA("Color3Value") then
					return ch.Value
				end
				if ch:IsA("BrickColorValue") then
					return ch.Value.Color
				end
			end
			local p = (inst:IsA("Model") and inst.PrimaryPart) or inst:FindFirstChildWhichIsA("BasePart", true)
			if p then
				return p.Color
			end
		end
		return nil
	end

	local function looksLikeTeamContainer(ln)
		return ln:find("teams", 1, true) ~= nil
			or ln:find("equipes", 1, true) ~= nil
			or ln:match("^team$") ~= nil
			or ln:match("^equipe$") ~= nil
			or ln:match("^times?$") ~= nil
			or ln:match("^times") ~= nil
	end

	-- Varre o Workspace (com pausas para não travar)
	local function scanWorkspace()
		local cache = { named = {}, spawns = {}, colored = {} }

		local known = {}
		local hasKnown = false
		for _, tm in ipairs(TeamsService:GetTeams()) do
			known[tm.Name:lower()] = tm.Name
			hasKnown = true
		end
		for _, plr in ipairs(Players:GetPlayers()) do
			local nm = playerTeamInfo(plr)
			if nm then
				known[nm:lower()] = nm
				hasKnown = true
			end
		end

		local function bucket(lname, display)
			local b = cache.named[lname]
			if not b then
				b = { name = display, count = 0, colors = {}, seen = {} }
				cache.named[lname] = b
			end
			return b
		end

		local function pushColor(b, c)
			if not c then
				return
			end
			local h = hex(c)
			if not b.seen[h] and #b.colors < 6 then
				b.seen[h] = true
				table.insert(b.colors, c)
			end
		end

		local function ownerOf(inst)
			local p = inst.Parent
			local d = 0
			while p and p ~= workspace and d < 5 do
				local ln = p.Name:lower()
				if known[ln] then
					return ln
				end
				p = p.Parent
				d += 1
			end
			return nil
		end

		local function processInst(inst)
			if inst:IsA("Terrain") then
				return
			end
			if inst:IsA("Model") and Players:GetPlayerFromCharacter(inst) then
				return
			end
			local lname = inst.Name:lower()

			-- SpawnLocation (TeamColor / Neutral)
			if inst:IsA("SpawnLocation") then
				local owner = ownerOf(inst)
				if owner or not inst.Neutral then
					table.insert(cache.spawns, {
						color = inst.TeamColor.Color,
						bcName = inst.TeamColor.Name,
						owner = owner,
					})
				end
				return
			end

			-- Contêineres que parecem agrupar times (só quando não há times conhecidos)
			if not hasKnown and (inst:IsA("Folder") or inst:IsA("Model") or inst:IsA("Configuration")) and looksLikeTeamContainer(lname) then
				for _, ch in ipairs(inst:GetChildren()) do
					if ch:IsA("Folder") or ch:IsA("Model") or ch:IsA("Configuration") or ch:IsA("Color3Value") or ch:IsA("BrickColorValue") then
						local b = bucket(ch.Name:lower(), ch.Name)
						b.count += 1
						pushColor(b, instColor(ch))
					end
				end
			end

			-- Atributo com nome do time
			local an = attrName(inst)
			if an then
				local b = bucket(an:lower(), an)
				b.count += 1
				pushColor(b, instColor(inst))
				return
			end

			-- Atributo TeamColor solto
			local tc = inst:GetAttribute("TeamColor")
			if typeof(tc) == "BrickColor" then
				table.insert(cache.colored, { color = tc.Color })
			elseif typeof(tc) == "Color3" then
				table.insert(cache.colored, { color = tc })
			end

			-- Estrutura cujo nome coincide com um time conhecido
			if known[lname] and (inst:IsA("PVInstance") or inst:IsA("Folder") or inst:IsA("Configuration") or inst:IsA("ValueBase")) then
				local b = bucket(lname, known[lname])
				b.count += 1
				pushColor(b, instColor(inst))
				return
			end

			-- Peças/valores dentro de estrutura com nome de time: coleta a cor
			if inst:IsA("BasePart") or inst:IsA("ValueBase") then
				local o = ownerOf(inst)
				if o then
					pushColor(bucket(o, known[o]), instColor(inst))
				end
			end
		end

		local list = workspace:GetDescendants()
		for i, inst in ipairs(list) do
			if i % 300 == 0 then
				task.wait()
				if destroyed or not S.DetectTeams then
					return cache
				end
			end
			if inst.Parent then
				pcall(processInst, inst)
			end
		end
		return cache
	end

	-- Monta a lista final de times (rápido; usa o cache do Workspace)
	local function buildTeams()
		local entries, order = {}, {}

		local function getEntry(key, name, source)
			local e = entries[key]
			if not e then
				e = {
					key = key,
					name = name,
					sources = {},
					color = nil,
					bcName = nil,
					structures = 0,
					colors = {},
					colorSeen = {},
					players = {},
				}
				entries[key] = e
				table.insert(order, e)
			end
			if source then
				local has = false
				for _, s in ipairs(e.sources) do
					if s == source then
						has = true
						break
					end
				end
				if not has then
					table.insert(e.sources, source)
				end
			end
			return e
		end

		local function addColor(e, c)
			local h = hex(c)
			if not e.colorSeen[h] and #e.colors < 6 then
				e.colorSeen[h] = true
				table.insert(e.colors, c)
			end
		end

		local function setColor(e, c, bcName)
			if c and not e.color then
				e.color = c
				e.bcName = bcName or BrickColor.new(c).Name
			end
		end

		local function findByColor(c)
			for _, e in ipairs(order) do
				if e.color and colorDist(e.color, c) < COLOR_TOLERANCE then
					return e
				end
			end
			return nil
		end

		-- 1) Times do serviço Teams
		for _, tm in ipairs(TeamsService:GetTeams()) do
			local e = getEntry(tm.Name:lower(), tm.Name, "Teams")
			e.team = tm
			e.color = tm.TeamColor.Color
			e.bcName = tm.TeamColor.Name
		end

		-- 2) Estruturas do Workspace (por nome/atributo)
		for lname, info in pairs(wsCache.named) do
			local e = getEntry(lname, info.name, "Workspace")
			e.structures += info.count
			for _, c in ipairs(info.colors) do
				addColor(e, c)
			end
			if info.colors[1] then
				setColor(e, info.colors[1])
			end
		end

		-- 3) Jogadores
		for _, plr in ipairs(Players:GetPlayers()) do
			local name, src, col = playerTeamInfo(plr)
			local e
			if name then
				e = getEntry(name:lower(), name, src)
				if col then
					setColor(e, col)
				end
			elseif not plr.Neutral then
				local c = plr.TeamColor.Color
				e = findByColor(c) or getEntry("color:" .. hex(c), plr.TeamColor.Name, "Cor do jogador")
				setColor(e, c, plr.TeamColor.Name)
				getEntry(e.key, e.name, "Cor do jogador")
			else
				e = getEntry("__none", "Sem time", nil)
				e.isNone = true
			end
			table.insert(e.players, plr)
		end

		-- 4) SpawnLocations relacionados por dono ou por cor
		for _, sp in ipairs(wsCache.spawns) do
			local e = sp.owner and entries[sp.owner] or nil
			if not e then
				e = findByColor(sp.color)
			end
			if not e then
				e = getEntry("color:" .. hex(sp.color), sp.bcName, "SpawnLocation")
			end
			getEntry(e.key, e.name, "SpawnLocation")
			e.structures += 1
			addColor(e, sp.color)
			setColor(e, sp.color, sp.bcName)
		end

		-- 5) Objetos com atributo TeamColor relacionados por cor
		for _, ci in ipairs(wsCache.colored) do
			local e = findByColor(ci.color)
			if e then
				e.structures += 1
				addColor(e, ci.color)
			end
		end

		-- Ordenação
		for _, e in ipairs(order) do
			table.sort(e.players, function(a, b)
				return a.Name:lower() < b.Name:lower()
			end)
		end
		table.sort(order, function(a, b)
			local an, bn = a.isNone and 1 or 0, b.isNone and 1 or 0
			if an ~= bn then
				return an < bn
			end
			local at, bt = a.team and 0 or 1, b.team and 0 or 1
			if at ~= bt then
				return at < bt
			end
			return a.name:lower() < b.name:lower()
		end)

		local totalPlayers, totalStructs = 0, 0
		for _, e in ipairs(order) do
			totalPlayers += #e.players
			totalStructs += e.structures
		end
		return order, totalPlayers, totalStructs
	end

	------------------------------------------------------------------
	-- Painel (janela flutuante com ScrollingFrame)
	------------------------------------------------------------------
	local panel = create("Frame", {
		Name = "TeamsPanel",
		Size = UDim2.fromOffset(300, 340),
		Position = UDim2.fromOffset(360, 96),
		BorderSizePixel = 0,
		ClipsDescendants = true,
		Active = true,
		Visible = false,
		Parent = gui,
	})
	reg(panel, "BackgroundColor3", "bg")
	regTransp(panel)
	corner(panel, 10)
	reg(create("UIStroke", { Thickness = 1, Parent = panel }), "Color", "off")
	local pScale = create("UIScale", { Scale = 1, Parent = panel })

	local pTitle = create("Frame", {
		Size = UDim2.new(1, 0, 0, 36),
		BorderSizePixel = 0,
		Parent = panel,
	})
	reg(pTitle, "BackgroundColor3", "title")
	regTransp(pTitle)

	reg(create("TextLabel", {
		BackgroundTransparency = 1,
		Position = UDim2.fromOffset(12, 0),
		Size = UDim2.new(1, -96, 1, 0),
		Text = "Times detectados",
		TextXAlignment = Enum.TextXAlignment.Left,
		Font = Enum.Font.GothamBold,
		TextSize = 14,
		Parent = pTitle,
	}), "TextColor3", "text")

	local refreshBtn = create("TextButton", {
		AnchorPoint = Vector2.new(1, 0.5),
		Position = UDim2.new(1, -44, 0.5, 0),
		Size = UDim2.fromOffset(34, 28),
		Text = "↻",
		Font = Enum.Font.GothamBold,
		TextSize = 16,
		AutoButtonColor = true,
		BorderSizePixel = 0,
		Parent = pTitle,
	})
	reg(refreshBtn, "BackgroundColor3", "panel")
	reg(refreshBtn, "TextColor3", "text")
	corner(refreshBtn, 6)

	local closeBtn2 = create("TextButton", {
		AnchorPoint = Vector2.new(1, 0.5),
		Position = UDim2.new(1, -6, 0.5, 0),
		Size = UDim2.fromOffset(34, 28),
		Text = "✕",
		Font = Enum.Font.GothamBold,
		TextSize = 14,
		AutoButtonColor = true,
		BorderSizePixel = 0,
		Parent = pTitle,
	})
	reg(closeBtn2, "BackgroundColor3", "panel")
	reg(closeBtn2, "TextColor3", "text")
	corner(closeBtn2, 6)

	local summary = create("TextLabel", {
		BackgroundTransparency = 1,
		Position = UDim2.fromOffset(12, 38),
		Size = UDim2.new(1, -24, 0, 20),
		Text = "Aguardando...",
		TextXAlignment = Enum.TextXAlignment.Left,
		TextTruncate = Enum.TextTruncate.AtEnd,
		Font = Enum.Font.GothamMedium,
		TextSize = 12,
		Parent = panel,
	})
	reg(summary, "TextColor3", "sub")

	local scroll = create("ScrollingFrame", {
		Position = UDim2.fromOffset(0, 60),
		Size = UDim2.new(1, 0, 1, -60),
		BackgroundTransparency = 1,
		BorderSizePixel = 0,
		ScrollBarThickness = 5,
		CanvasSize = UDim2.new(0, 0, 0, 0),
		AutomaticCanvasSize = Enum.AutomaticSize.Y,
		ScrollingDirection = Enum.ScrollingDirection.Y,
		Parent = panel,
	})
	reg(scroll, "ScrollBarImageColor3", "accent")
	create("UIListLayout", { Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder, Parent = scroll })
	create("UIPadding", {
		PaddingTop = UDim.new(0, 2),
		PaddingBottom = UDim.new(0, 10),
		PaddingLeft = UDim.new(0, 8),
		PaddingRight = UDim.new(0, 10),
		Parent = scroll,
	})

	local positioned = false
	local function clampPanel()
		local cam = workspace.CurrentCamera
		local vp = cam and cam.ViewportSize or Vector2.new(800, 600)
		local p = panel.Position
		local x = math.clamp(p.X.Offset, 0, math.max(vp.X - 60, 0))
		local y = math.clamp(p.Y.Offset, 0, math.max(vp.Y - 80, 0))
		panel.Position = UDim2.fromOffset(x, y)
	end

	function teamsApi.layout()
		local cam = workspace.CurrentCamera
		local vp = cam and cam.ViewportSize or Vector2.new(800, 600)
		local sc = S.Scale
		local w = math.clamp((vp.X - 24) / sc, 200, 320)
		local h = math.clamp((vp.Y - 130) / sc, 160, 380)
		panel.Size = UDim2.fromOffset(w, h)
		pScale.Scale = sc
		if not positioned then
			positioned = true
			panel.Position = UDim2.fromOffset(math.max(vp.X - w * sc - 8, 8), 96)
		end
		clampPanel()
	end

	-- Arrastar o painel pela barra de título
	do
		local startPos, startMouse
		bindDrag(pTitle, function(input)
			if not S.Draggable then
				return false
			end
			startPos = panel.Position
			startMouse = Vector2.new(input.Position.X, input.Position.Y)
			return true
		end, function(input)
			if not startPos then
				return
			end
			local cam = workspace.CurrentCamera
			local vp = cam and cam.ViewportSize or Vector2.new(800, 600)
			local dx = input.Position.X - startMouse.X
			local dy = input.Position.Y - startMouse.Y
			local x = math.clamp(startPos.X.Offset + dx, 0, math.max(vp.X - 60, 0))
			local y = math.clamp(startPos.Y.Offset + dy, 0, math.max(vp.Y - 80, 0))
			panel.Position = UDim2.fromOffset(x, y)
		end)
	end

	------------------------------------------------------------------
	-- Renderização
	------------------------------------------------------------------
	local function playerLine(plr)
		local nm = plr.Name
		if plr.DisplayName ~= "" and plr.DisplayName ~= plr.Name then
			nm = plr.DisplayName .. " (@" .. plr.Name .. ")"
		end
		if plr == LocalPlayer then
			nm = nm .. " [você]"
		end
		return "• " .. nm
	end

	local function buildCard(e, order)
		local th = T()
		local card = create("Frame", {
			Size = UDim2.new(1, 0, 0, 0),
			AutomaticSize = Enum.AutomaticSize.Y,
			BackgroundColor3 = th.panel,
			BackgroundTransparency = S.Transparency,
			BorderSizePixel = 0,
			LayoutOrder = order,
			Parent = scroll,
		})
		corner(card, 8)
		create("UIListLayout", { Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder, Parent = card })
		create("UIPadding", {
			PaddingTop = UDim.new(0, 8),
			PaddingBottom = UDim.new(0, 8),
			PaddingLeft = UDim.new(0, 10),
			PaddingRight = UDim.new(0, 10),
			Parent = card,
		})

		-- Cabeçalho: cor + nome + contagem
		local head = create("Frame", {
			Size = UDim2.new(1, 0, 0, 24),
			BackgroundTransparency = 1,
			LayoutOrder = 1,
			Parent = card,
		})
		local sw = create("Frame", {
			AnchorPoint = Vector2.new(0, 0.5),
			Position = UDim2.new(0, 0, 0.5, 0),
			Size = UDim2.fromOffset(20, 20),
			BackgroundColor3 = e.color or th.off,
			BorderSizePixel = 0,
			Parent = head,
		})
		corner(sw, 5)
		create("UIStroke", { Color = th.text, Transparency = 0.5, Thickness = 1, Parent = sw })
		create("TextLabel", {
			BackgroundTransparency = 1,
			Position = UDim2.fromOffset(28, 0),
			Size = UDim2.new(1, -78, 1, 0),
			Text = e.name,
			TextXAlignment = Enum.TextXAlignment.Left,
			TextTruncate = Enum.TextTruncate.AtEnd,
			Font = Enum.Font.GothamBold,
			TextSize = 15,
			TextColor3 = th.text,
			Parent = head,
		})
		create("TextLabel", {
			BackgroundTransparency = 1,
			AnchorPoint = Vector2.new(1, 0),
			Position = UDim2.new(1, 0, 0, 0),
			Size = UDim2.fromOffset(46, 24),
			Text = tostring(#e.players),
			TextXAlignment = Enum.TextXAlignment.Right,
			Font = Enum.Font.GothamBold,
			TextSize = 14,
			TextColor3 = th.accent,
			Parent = head,
		})

		-- Informações
		local info = {}
		if e.color then
			table.insert(info, string.format("Cor: %s (%s)", hex(e.color), e.bcName or "?"))
		elseif not e.isNone then
			table.insert(info, "Cor: não encontrada")
		end
		if e.structures > 0 then
			table.insert(info, "Estruturas relacionadas: " .. e.structures)
		end
		if #e.sources > 0 then
			table.insert(info, "Fonte: " .. table.concat(e.sources, ", "))
		end
		if #info > 0 then
			create("TextLabel", {
				Size = UDim2.new(1, 0, 0, 0),
				AutomaticSize = Enum.AutomaticSize.Y,
				BackgroundTransparency = 1,
				TextWrapped = true,
				TextXAlignment = Enum.TextXAlignment.Left,
				TextYAlignment = Enum.TextYAlignment.Top,
				Font = Enum.Font.Gotham,
				TextSize = 12,
				TextColor3 = th.sub,
				Text = table.concat(info, "\n"),
				LayoutOrder = 2,
				Parent = card,
			})
		end

		-- Cores encontradas em estruturas
		if #e.colors > 0 then
			local row = create("Frame", {
				Size = UDim2.new(1, 0, 0, 18),
				BackgroundTransparency = 1,
				LayoutOrder = 3,
				Parent = card,
			})
			create("UIListLayout", {
				FillDirection = Enum.FillDirection.Horizontal,
				Padding = UDim.new(0, 4),
				VerticalAlignment = Enum.VerticalAlignment.Center,
				SortOrder = Enum.SortOrder.LayoutOrder,
				Parent = row,
			})
			create("TextLabel", {
				Size = UDim2.fromOffset(0, 18),
				AutomaticSize = Enum.AutomaticSize.X,
				BackgroundTransparency = 1,
				Font = Enum.Font.Gotham,
				TextSize = 12,
				TextColor3 = th.sub,
				Text = "Cores em estruturas:",
				LayoutOrder = 0,
				Parent = row,
			})
			for i, c in ipairs(e.colors) do
				local s = create("Frame", {
					Size = UDim2.fromOffset(16, 16),
					BackgroundColor3 = c,
					BorderSizePixel = 0,
					LayoutOrder = i,
					Parent = row,
				})
				corner(s, 4)
				create("UIStroke", { Color = th.text, Transparency = 0.6, Thickness = 1, Parent = s })
			end
		end

		-- Jogadores
		local txt
		if #e.players == 0 then
			txt = "Nenhum jogador"
		else
			local lines = { "Jogadores (" .. #e.players .. "):" }
			for _, p in ipairs(e.players) do
				table.insert(lines, playerLine(p))
			end
			txt = table.concat(lines, "\n")
		end
		create("TextLabel", {
			Size = UDim2.new(1, 0, 0, 0),
			AutomaticSize = Enum.AutomaticSize.Y,
			BackgroundTransparency = 1,
			TextWrapped = true,
			TextXAlignment = Enum.TextXAlignment.Left,
			TextYAlignment = Enum.TextYAlignment.Top,
			Font = Enum.Font.GothamMedium,
			TextSize = 13,
			TextColor3 = th.text,
			Text = txt,
			LayoutOrder = 4,
			Parent = card,
		})
	end

	local function renderTeams(force)
		if not S.DetectTeams or destroyed then
			return
		end
		local list, totalPlayers, totalStructs = buildTeams()
		local realTeams = 0
		for _, e in ipairs(list) do
			if not e.isNone then
				realTeams += 1
			end
		end
		summary.Text = string.format("%d times | %d jogadores | %d estruturas", realTeams, totalPlayers, totalStructs)

		-- Assinatura para evitar reconstruir sem necessidade
		local parts = {}
		for _, e in ipairs(list) do
			local names, cols = {}, {}
			for _, p in ipairs(e.players) do
				names[#names + 1] = p.Name .. ":" .. p.DisplayName
			end
			for _, c in ipairs(e.colors) do
				cols[#cols + 1] = hex(c)
			end
			parts[#parts + 1] = table.concat({
				e.key,
				e.name,
				e.color and hex(e.color) or "-",
				e.bcName or "-",
				tostring(e.structures),
				table.concat(cols, ","),
				table.concat(e.sources, ","),
				table.concat(names, ","),
			}, "|")
		end
		local sig = table.concat(parts, "\n") .. "#" .. S.Theme .. "#" .. tostring(S.Transparency)
		if not force and sig == lastSig then
			return
		end
		lastSig = sig

		local pos = scroll.CanvasPosition
		for _, c in ipairs(scroll:GetChildren()) do
			if c:IsA("GuiObject") then
				c:Destroy()
			end
		end

		if #list == 0 then
			local th = T()
			local card = create("Frame", {
				Size = UDim2.new(1, 0, 0, 48),
				BackgroundColor3 = th.panel,
				BackgroundTransparency = S.Transparency,
				BorderSizePixel = 0,
				LayoutOrder = 1,
				Parent = scroll,
			})
			corner(card, 8)
			create("TextLabel", {
				Size = UDim2.fromScale(1, 1),
				BackgroundTransparency = 1,
				Text = "Nenhum time detectado neste jogo",
				Font = Enum.Font.GothamMedium,
				TextSize = 13,
				TextColor3 = th.sub,
				Parent = card,
			})
		else
			for i, e in ipairs(list) do
				buildCard(e, i)
			end
		end

		scroll.CanvasPosition = pos
		task.delay(0.05, function()
			if scroll.Parent then
				scroll.CanvasPosition = pos
			end
		end)
	end
	teamsApi.render = renderTeams

	------------------------------------------------------------------
	-- Agendamento / atualização automática
	------------------------------------------------------------------
	local pending, scanning, wsDirty = false, false, true

	local function requestTeams(full)
		if full then
			wsDirty = true
		end
		if destroyed or not S.DetectTeams or pending then
			return
		end
		pending = true
		task.delay(0.25, function()
			pending = false
			if destroyed or not S.DetectTeams then
				return
			end
			if wsDirty and not scanning then
				wsDirty = false
				scanning = true
				summary.Text = "Escaneando o Workspace..."
				local ok, res = pcall(scanWorkspace)
				scanning = false
				if ok and res then
					wsCache = res
				end
				if destroyed or not S.DetectTeams then
					return
				end
				if wsDirty then
					pending = false
					task.defer(function()
						requestTeams(false)
					end)
				end
			end
			local ok2, err = pcall(renderTeams, false)
			if not ok2 then
				warn("[MenuUtil30] Erro em Detectar Times: " .. tostring(err))
			end
		end)
	end

	-- Eventos de jogadores
	local function hookPlayer(plr)
		for _, prop in ipairs({ "Team", "TeamColor", "Neutral" }) do
			connect(plr:GetPropertyChangedSignal(prop), function()
				requestTeams(false)
			end)
		end
		connect(plr.AttributeChanged, function()
			requestTeams(false)
		end)
	end
	for _, p in ipairs(Players:GetPlayers()) do
		hookPlayer(p)
	end
	connect(Players.PlayerAdded, function(p)
		hookPlayer(p)
		requestTeams(true)
	end)
	connect(Players.PlayerRemoving, function()
		task.defer(function()
			requestTeams(true)
		end)
	end)

	-- Eventos do serviço Teams
	local function hookTeam(tm)
		if not tm:IsA("Team") then
			return
		end
		connect(tm:GetPropertyChangedSignal("TeamColor"), function()
			requestTeams(true)
		end)
		connect(tm:GetPropertyChangedSignal("Name"), function()
			requestTeams(true)
		end)
		connect(tm.PlayerAdded, function()
			requestTeams(false)
		end)
		connect(tm.PlayerRemoved, function()
			requestTeams(false)
		end)
	end
	for _, tm in ipairs(TeamsService:GetTeams()) do
		hookTeam(tm)
	end
	connect(TeamsService.ChildAdded, function(c)
		hookTeam(c)
		requestTeams(true)
	end)
	connect(TeamsService.ChildRemoved, function()
		requestTeams(true)
	end)

	-- Eventos do Workspace (somente enquanto a opção estiver ligada)
	local function connectWs()
		if #wsConns > 0 then
			return
		end
		table.insert(wsConns, workspace.DescendantAdded:Connect(function(d)
			if S.DetectTeams and d:IsA("SpawnLocation") then
				requestTeams(true)
			end
		end))
		table.insert(wsConns, workspace.DescendantRemoving:Connect(function(d)
			if S.DetectTeams and d:IsA("SpawnLocation") then
				requestTeams(true)
			end
		end))
	end
	local function disconnectWs()
		for _, c in ipairs(wsConns) do
			pcall(function()
				c:Disconnect()
			end)
		end
		wsConns = {}
	end

	-- Atualização periódica (pega mudanças que não geram evento)
	task.spawn(function()
		local t = 0
		while not destroyed do
			task.wait(1)
			if S.DetectTeams then
				t += 1
				if t % 15 == 0 then
					requestTeams(true)
				elseif t % 3 == 0 then
					requestTeams(false)
				end
			end
		end
	end)

	connect(refreshBtn.Activated, function()
		playClick()
		requestTeams(true)
	end)
	connect(closeBtn2.Activated, function()
		playClick()
		setSetting("DetectTeams", false)
	end)

	handlers.DetectTeams = function(v)
		panel.Visible = v
		if v then
			teamsApi.layout()
			connectWs()
			lastSig = nil
			summary.Text = "Escaneando o Workspace..."
			if S.Animations then
				pScale.Scale = S.Scale * 0.92
				tween(pScale, { Scale = S.Scale }, 0.15)
			end
			requestTeams(true)
		else
			disconnectWs()
		end
	end

	function teamsApi.cleanup()
		destroyed = true
		disconnectWs()
	end
end

--==================================================================
-- Handlers (aplicam cada opção no jogo)
--==================================================================
handlers.WalkSpeed = function(v)
	local h = getHum()
	if h then
		h.WalkSpeed = v
	end
end

handlers.JumpPower = function(v)
	local h = getHum()
	if not h then
		return
	end
	if v == D.JumpPower then
		h.UseJumpPower = origUseJump
		h.JumpPower = D.JumpPower
	else
		h.UseJumpPower = true
		h.JumpPower = v
	end
end

handlers.AutoRotate = function(v)
	local h = getHum()
	if h and not S.ShiftLock then
		h.AutoRotate = v
	end
end

handlers.ShiftLock = function(v)
	local h = getHum()
	if h then
		if v then
			h.AutoRotate = false
		else
			h.AutoRotate = S.AutoRotate
			h.CameraOffset = Vector3.zero
		end
	end
	if not v then
		shiftHum = nil
	end
end

handlers.FOV = function(v)
	local cam = workspace.CurrentCamera
	if cam then
		cam.FieldOfView = v
	end
end

handlers.MaxZoom = applyZoom
handlers.MinZoom = applyZoom

handlers.CamSens = function(v)
	if UGS then
		pcall(function()
			UGS.GamepadCameraSensitivity = v
		end)
	end
end

handlers.MouseSens = function(v)
	if UGS then
		pcall(function()
			UGS.MouseSensitivity = v
		end)
	end
end

handlers.ShowNames = refreshESP
handlers.ShowDistance = refreshESP
handlers.Highlight = refreshESP
handlers.Quick_Noclip = function(_)
	-- Quick_Noclip controla apenas a disponibilidade do botão no Menu Rápido.
end
handlers.Quick_ESP = function(_)
	-- Quick_ESP controla a disponibilidade do botão no Menu Rápido.
	-- O botão "ESP" usa setESPAll() para ligar/desligar os três recursos.
end
handlers.FlyEnabled = function(v)
	setFlyEnabled(v)
end
handlers.Noclip = function(v)
	local character = LocalPlayer.Character
	if not character then return end
	for _, part in ipairs(character:GetDescendants()) do
		if part:IsA("BasePart") then
			pcall(function() part.CanCollide = not v end)
		end
	end
end

handlers.FlyFrozen = function(v)
	setFlyFrozen(v)
end
handlers.FlySpeed = function(v)
	setFlySpeed(v)
end
handlers.PlayerCount = applyHudVisibility
handlers.ShowFPS = applyHudVisibility
handlers.ShowPing = applyHudVisibility
handlers.ShowCoords = applyHudVisibility

handlers.InterfaceOn = function(v)
	Main.Visible = v
	OpenBtn.Visible = not v
	if v and S.Animations then
		uiScale.Scale = S.Scale * 0.92
		tween(uiScale, { Scale = S.Scale }, 0.15)
	end
end

handlers.Minimized = function(v)
	Body.Visible = not v
	MinBtn.Text = v and "+" or "–"
	updateLayout(true)
end

handlers.Transparency = applyTransparency

handlers.Scale = function()
	updateLayout(false)
end

handlers.Theme = function()
	applyTheme()
	refreshAll(true)
	refreshESP()
	if teamsApi.render then
		teamsApi.render(true)
	end
end

--==================================================================
-- MENU RÁPIDO
--==================================================================
local quickFrame = create("Frame", {
	Name = "MenuRapido",
	Size = UDim2.fromOffset(180, 0),
	AutomaticSize = Enum.AutomaticSize.Y,
	Position = UDim2.fromOffset(350, 120),
	BackgroundTransparency = 0.05,
	BorderSizePixel = 0,
	Active = true,
	Parent = gui,
})
reg(quickFrame, "BackgroundColor3", "bg")
corner(quickFrame, 10)
local quickTitle = create("TextButton", {
	Size = UDim2.new(1, 0, 0, 30),
	BackgroundTransparency = 1,
	Text = "MENU RÁPIDO  ▲",
	Font = Enum.Font.GothamBold,
	TextSize = 13,
	TextColor3 = Color3.new(1,1,1),
	AutoButtonColor = false,
	Parent = quickFrame,
})
local quickList = create("Frame", {
	Position = UDim2.fromOffset(7, 34),
	Size = UDim2.new(1, -14, 0, 0),
	AutomaticSize = Enum.AutomaticSize.Y,
	BackgroundTransparency = 1,
	Parent = quickFrame,
})
create("UIListLayout", { Padding = UDim.new(0, 5), SortOrder = Enum.SortOrder.LayoutOrder, Parent = quickList })

local quickMenuOpen = true
local quickDragHandle = create("TextButton", {
	Name = "DragHandle", Position = UDim2.fromOffset(4, 4), Size = UDim2.fromOffset(28, 22),
	Text = "⋮⋮", Font = Enum.Font.GothamBold, TextSize = 14, TextColor3 = Color3.new(1,1,1),
	BackgroundTransparency = 0.35, BorderSizePixel = 0, AutoButtonColor = false, Parent = quickFrame,
})
reg(quickDragHandle, "BackgroundColor3", "panel"); corner(quickDragHandle, 5)
quickTitle.Position = UDim2.fromOffset(32, 0)
quickTitle.Size = UDim2.new(1, -32, 0, 30)
connect(quickTitle.Activated, function()
	quickMenuOpen = not quickMenuOpen
	quickList.Visible = quickMenuOpen
	quickTitle.Text = quickMenuOpen and "MENU RÁPIDO  ▲" or "MENU RÁPIDO  ▼"
end)

local quickDefs = {
	Fly = {label = "✈ Fly", enabled = "Quick_Fly"},
	Noclip = {label = "🚫 Noclip", enabled = "Quick_Noclip"},
	ESP = {label = "👁 ESP", enabled = "Quick_ESP"},
	SavePosition = {label = "📍 Salvar posição", enabled = "Quick_SavePosition"},
	GotoPosition = {label = "➜ Ir para posição", enabled = "Quick_GotoPosition"},
	VerTimes = {label = "👥 Ver times", enabled = "Quick_VerTimes"},
	ShowNames = {label = "Nome", enabled = "Quick_ShowNames"},
	ShowDistance = {label = "Distância", enabled = "Quick_ShowDistance"},
	Highlight = {label = "Highlight", enabled = "Quick_Highlight"},
}
local quickButtons = {}
local quickConns = {}
local function clearQuickConnections()
	for _, c in ipairs(quickConns) do pcall(function() c:Disconnect() end) end
	quickConns = {}
end

local function quickAction(id)
	if id == "Fly" then
		toggleFly()
	elseif id == "Noclip" then
		setSetting("Noclip", not S.Noclip)
	elseif id == "ESP" then
		local espOn = S.Highlight or S.ShowNames or S.ShowDistance
		setESPAll(not espOn)
	elseif id == "SavePosition" then
		savePosition()
	elseif id == "GotoPosition" then
		goToSavedPosition()
	elseif id == "VerTimes" then
		setSetting("DetectTeams", true)
	elseif id == "ShowNames" then
		setSetting("ShowNames", not S.ShowNames)
	elseif id == "ShowDistance" then
		setSetting("ShowDistance", not S.ShowDistance)
	elseif id == "Highlight" then
		setSetting("Highlight", not S.Highlight)
	end
end

local function rebuildQuickMenu()
	clearQuickConnections()
	for _, b in pairs(quickButtons) do b:Destroy() end
	quickButtons = {}
	for orderIndex, id in ipairs(S.QuickOrder) do
		local def = quickDefs[id]
		if def and S[def.enabled] then
			local b = create("TextButton", {
				Size = UDim2.new(1, 0, 0, 32),
				Text = def.label,
				Font = Enum.Font.GothamBold,
				TextSize = 12,
				TextColor3 = Color3.new(1,1,1),
				BorderSizePixel = 0,
				AutoButtonColor = true,
				LayoutOrder = orderIndex,
				Parent = quickList,
			})
			reg(b, "BackgroundColor3", "panel")
			corner(b, 7)
			quickButtons[id] = b
			table.insert(quickConns, b.Activated:Connect(function()
				playClick()
				quickAction(id)
			end))
		end
	end
end

local quickEditor = create("Frame", {
	Name = "QuickEditor",
	Size = UDim2.fromOffset(250, 0),
	AutomaticSize = Enum.AutomaticSize.Y,
	Position = UDim2.fromOffset(350, 300),
	BackgroundTransparency = 0.05,
	BorderSizePixel = 0,
	Visible = false,
	Active = true,
	Parent = gui,
})
reg(quickEditor, "BackgroundColor3", "bg")
corner(quickEditor, 10)
create("TextLabel", {
	Size = UDim2.new(1,0,0,34),
	BackgroundTransparency = 1,
	Text = "Editar Menu Rápido",
	Font = Enum.Font.GothamBold,
	TextSize = 14,
	TextColor3 = Color3.new(1,1,1),
	Parent = quickEditor,
})
local quickEditorList = create("Frame", {
	Position = UDim2.fromOffset(8, 38),
	Size = UDim2.new(1,-16,0,0),
	AutomaticSize = Enum.AutomaticSize.Y,
	BackgroundTransparency = 1,
	Parent = quickEditor,
})
create("UIListLayout", {Padding = UDim.new(0,4), SortOrder = Enum.SortOrder.LayoutOrder, Parent = quickEditorList})
local quickEditorRows = {}
local quickEditorConns = {}
local function clearQuickEditorConnections()
	for _, c in ipairs(quickEditorConns) do pcall(function() c:Disconnect() end) end
	quickEditorConns = {}
end

local rebuildQuickEditor

local function moveQuickItem(id, delta)
	local order = S.QuickOrder
	local index
	for i, value in ipairs(order) do
		if value == id then
			index = i
			break
		end
	end
	if not index then return end
	local newIndex = index + delta
	if newIndex < 1 or newIndex > #order then return end
	order[index], order[newIndex] = order[newIndex], order[index]
	rebuildQuickMenu()
	rebuildQuickEditor()
	pcall(saveConfig)
end

rebuildQuickEditor = function()
	clearQuickEditorConnections()
	for _, row in pairs(quickEditorRows) do row:Destroy() end
	quickEditorRows = {}

	for index, id in ipairs(S.QuickOrder) do
		local def = quickDefs[id]
		if def then
			local row = create("Frame", {
				Size = UDim2.new(1,0,0,34),
				BackgroundTransparency = 1,
				LayoutOrder = index,
				Parent = quickEditorList,
			})
			local toggle = create("TextButton", {
				Size = UDim2.new(1,-72,1,0),
				Text = (S[def.enabled] and "☑ " or "☐ ") .. def.label,
				Font = Enum.Font.GothamMedium,
				TextSize = 12,
				TextXAlignment = Enum.TextXAlignment.Left,
				TextColor3 = Color3.new(1,1,1),
				BorderSizePixel = 0,
				AutoButtonColor = true,
				Parent = row,
			})
			reg(toggle, "BackgroundColor3", "panel")
			corner(toggle, 7)
			local up = create("TextButton", {
				Position = UDim2.new(1,-68,0,0), Size = UDim2.fromOffset(32,34),
				Text = "↑", Font = Enum.Font.GothamBold, TextSize = 16,
				TextColor3 = Color3.new(1,1,1), BorderSizePixel = 0, Parent = row,
			})
			reg(up, "BackgroundColor3", "off"); corner(up, 7)
			local down = create("TextButton", {
				Position = UDim2.new(1,-34,0,0), Size = UDim2.fromOffset(32,34),
				Text = "↓", Font = Enum.Font.GothamBold, TextSize = 16,
				TextColor3 = Color3.new(1,1,1), BorderSizePixel = 0, Parent = row,
			})
			reg(down, "BackgroundColor3", "off"); corner(down, 7)
			quickEditorRows[id] = row
			table.insert(quickEditorConns, toggle.Activated:Connect(function()
				playClick()
				S[def.enabled] = not S[def.enabled]
				rebuildQuickMenu()
				rebuildQuickEditor()
				pcall(saveConfig)
			end))
			table.insert(quickEditorConns, up.Activated:Connect(function()
				playClick(); moveQuickItem(id, -1)
			end))
			table.insert(quickEditorConns, down.Activated:Connect(function()
				playClick(); moveQuickItem(id, 1)
			end))
		end
	end

	local reset = create("TextButton", {
		Size = UDim2.new(1,0,0,32), Text = "↺ Restaurar Menu Rápido",
		Font = Enum.Font.GothamBold, TextSize = 12, TextColor3 = Color3.new(1,1,1),
		BorderSizePixel = 0, LayoutOrder = #S.QuickOrder + 1, Parent = quickEditorList,
	})
	reg(reset, "BackgroundColor3", "accent"); corner(reset, 7)
	table.insert(quickEditorConns, reset.Activated:Connect(function()
		playClick()
		S.QuickOrder = {"Fly","Noclip","ESP","SavePosition","GotoPosition","VerTimes","ShowNames","ShowDistance","Highlight"}
		S.Quick_Fly = true; S.Quick_Noclip = false; S.Quick_ESP = true; S.Quick_SavePosition = false; S.Quick_GotoPosition = true
		S.Quick_VerTimes = false; S.Quick_ShowNames = false; S.Quick_ShowDistance = false; S.Quick_Highlight = false
		rebuildQuickMenu(); rebuildQuickEditor(); pcall(saveConfig)
		notify("Menu Rápido restaurado", true)
	end))
end


do
	local dragging = false
	local dragInput = nil
	local startPos = nil
	local startPoint = nil

	connect(quickDragHandle.InputBegan, function(input)
		if not S.Draggable then return end
		local t = input.UserInputType
		if t == Enum.UserInputType.MouseButton1 or t == Enum.UserInputType.Touch then
			dragging = true
			dragInput = input
			startPos = quickFrame.Position
			startPoint = Vector2.new(input.Position.X, input.Position.Y)
		end
	end)

	connect(UserInputService.InputChanged, function(input)
		if not dragging then return end
		if input.UserInputType ~= Enum.UserInputType.MouseMovement and input.UserInputType ~= Enum.UserInputType.Touch then return end
		local cam = workspace.CurrentCamera
		local vp = cam and cam.ViewportSize or Vector2.new(800, 600)
		local dx = input.Position.X - startPoint.X
		local dy = input.Position.Y - startPoint.Y
		quickFrame.Position = UDim2.fromOffset(
			math.clamp(startPos.X.Offset + dx, 0, math.max(vp.X - quickFrame.AbsoluteSize.X, 0)),
			math.clamp(startPos.Y.Offset + dy, 0, math.max(vp.Y - quickFrame.AbsoluteSize.Y, 0))
		)
	end)

	connect(UserInputService.InputEnded, function(input)
		if input == dragInput or input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = false
			dragInput = nil
		end
	end)
end

do
	local startPos, startMouse
	bindDrag(quickEditor, function(input)
		if not S.Draggable then return false end
		startPos = quickEditor.Position
		startMouse = Vector2.new(input.Position.X, input.Position.Y)
		return true
	end, function(input)
		if not startPos then return end
		local cam = workspace.CurrentCamera
		local vp = cam and cam.ViewportSize or Vector2.new(800,600)
		quickEditor.Position = UDim2.fromOffset(
			math.clamp(startPos.X.Offset + input.Position.X - startMouse.X, 0, math.max(vp.X - 260, 0)),
			math.clamp(startPos.Y.Offset + input.Position.Y - startMouse.Y, 0, math.max(vp.Y - 120, 0))
		)
	end)
end

rebuildQuickMenu()
rebuildQuickEditor()

--==================================================================
-- SISTEMA DE ABAS
--==================================================================
local activeTab = "Geral"
local tabButtons = {}
local tabGroups = {
	Geral = {"PERSONAGEM","CÂMERA","DESEMPENHO","UTILIDADES","INTERFACE","CONFIGURAÇÕES"},
	MOVIMENTO = {"MOVIMENTO"},
	["JOGADORES / ESP"] = {"JOGADORES / ESP","TIMES"},
}

local function setActiveTab(tab)
	activeTab = tab
	for name, button in pairs(tabButtons) do
		button.BackgroundColor3 = name == tab and T().accent or T().off
	end
	for categoryName, wrap in pairs(categoryRegistry) do
		local visible = false
		if tab == "Geral" then
			for _, n in ipairs(tabGroups.Geral) do if n == categoryName then visible = true break end end
		elseif tab == "MOVIMENTO" then
			visible = categoryName == "MOVIMENTO"
		elseif tab == "JOGADORES / ESP" then
			visible = categoryName == "JOGADORES / ESP" or categoryName == "TIMES"
		end
		wrap.Visible = visible
	end
end

for _, name in ipairs({"Geral","MOVIMENTO","JOGADORES / ESP"}) do
	local b = create("TextButton", {
		Size = UDim2.new(1/3, -4, 1, 0),
		Text = name,
		Font = Enum.Font.GothamBold,
		TextSize = 11,
		TextColor3 = Color3.new(1,1,1),
		BorderSizePixel = 0,
		AutoButtonColor = false,
		Parent = TabBar,
	})
	corner(b, 6)
	tabButtons[name] = b
	connect(b.Activated, function()
		playClick()
		setActiveTab(name)
	end)
end

-- Botões da barra de título e botão flutuante
connect(MinBtn.Activated, function()
	playClick()
	setSetting("Minimized", not S.Minimized, true)
	if refreshers.Minimized then
		refreshers.Minimized(false)
	end
end)

connect(CloseBtn.Activated, function()
	playClick()
	setSetting("InterfaceOn", false)
end)

connect(OpenBtn.Activated, function()
	playClick()
	setSetting("InterfaceOn", true)
end)

-- Atalho de teclado (PC)
connect(UserInputService.InputBegan, function(input, gameProcessed)
	if gameProcessed then
		return
	end
	if input.KeyCode == Enum.KeyCode.RightControl then
		setSetting("InterfaceOn", not S.InterfaceOn, true)
	end
end)

-- Ajuste ao mudar o tamanho da tela
do
	local cam = workspace.CurrentCamera
	if cam then
		connect(cam:GetPropertyChangedSignal("ViewportSize"), function()
			updateLayout(false)
		end)
	end
end

connect(LocalPlayer.CharacterAdded, function(character)
	task.wait(0.25)
	if S.Noclip then
		for _, part in ipairs(character:GetDescendants()) do
			if part:IsA("BasePart") then pcall(function() part.CanCollide = false end) end
		end
	end
end)

--==================================================================
-- Loops (Heartbeat)
--==================================================================
do
	local frames, hudAcc, espAcc, slowAcc = 0, 0, 0, 0

	connect(RunService.RenderStepped, function()
		frames += 1
	end)

	connect(RunService.Heartbeat, function(dt)
		updateFly(dt)
		local hum, root = getHum(), getRoot()
		if hum then
			enforceCharacter(hum)
			if S.Noclip then
				local character = LocalPlayer.Character
				if character then
					for _, part in ipairs(character:GetDescendants()) do
						if part:IsA("BasePart") and part.CanCollide then pcall(function() part.CanCollide = false end) end
					end
				end
			end
			if root and hum.Health > 0 then
				lastAliveCF = root.CFrame
			end
		end

		hudAcc += dt
		if hudAcc >= 0.25 then
			lastFps = frames / hudAcc
			frames = 0
			hudAcc = 0
			if HudFrame.Visible then
				updateHud()
			end
		end

		espAcc += dt
		if espAcc >= 0.2 then
			espAcc = 0
			if S.Highlight or S.ShowNames or S.ShowDistance or next(esp) ~= nil then
				refreshESP()
			end
		end

		slowAcc += dt
		if slowAcc >= 1 then
			slowAcc = 0
			if S.MaxZoom ~= D.MaxZoom or S.MinZoom ~= D.MinZoom then
				local mx = math.max(S.MaxZoom, S.MinZoom)
				if LocalPlayer.CameraMaxZoomDistance ~= mx or LocalPlayer.CameraMinZoomDistance ~= S.MinZoom then
					applyZoom()
				end
			end
		end
	end)
end

--==================================================================
-- Limpeza (restaura o jogo ao original ao reexecutar / remover)
--==================================================================
local function restoreEnvironment()
	local character = LocalPlayer.Character
	if character then
		for _, part in ipairs(character:GetDescendants()) do
			if part:IsA("BasePart") then pcall(function() part.CanCollide = true end) end
		end
	end
	local root = getRoot()
	if root then pcall(function() root.Anchored = false end) end
	local hum = getHum()
	if hum then
		pcall(function()
			hum.WalkSpeed = D.WalkSpeed
			hum.UseJumpPower = origUseJump
			hum.JumpPower = D.JumpPower
			hum.AutoRotate = D.AutoRotate
			hum.CameraOffset = Vector3.zero
		end)
	end
	pcall(function()
		workspace.CurrentCamera.FieldOfView = D.FOV
	end)
	pcall(function()
		LocalPlayer.CameraMaxZoomDistance = D.MaxZoom
		LocalPlayer.CameraMinZoomDistance = D.MinZoom
	end)
	if UGS then
		pcall(function()
			UGS.GamepadCameraSensitivity = D.CamSens
			UGS.MouseSensitivity = D.MouseSens
		end)
	end
end

shared.MenuUtil30_Cleanup = function()
	if teamsApi.cleanup then
		pcall(teamsApi.cleanup)
	end
	for _, c in ipairs(conns) do
		pcall(function()
			c:Disconnect()
		end)
	end
	pcall(function()
		RunService:UnbindFromRenderStep("MenuUtil30_Render")
	end)
	for plr in pairs(esp) do
		clearESP(plr)
	end
	restoreEnvironment()
	pcall(function()
		gui:Destroy()
	end)
	pcall(function()
		clickSound:Destroy()
	end)
end

--==================================================================
-- Aplicação inicial
--==================================================================
applyTheme()
applyTransparency()
updateLayout(false)
refreshAll(true)
applyHudVisibility()
setActiveTab("Geral")
rebuildQuickMenu()
rebuildQuickEditor()

for k, h in pairs(handlers) do
	if S[k] ~= D[k] then
		pcall(h, S[k])
	end
end

notify("Menu carregado")

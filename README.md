-- Services
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local Workspace = game:GetService("Workspace")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer

-- Registro de conexiones y cierre seguro.
local closed = false
local connections = {}
local playerConnections = {}
local updatePlayerList

local function connect(signal, callback)
    local connection = signal:Connect(function(...)
        if not closed then
            callback(...)
        end
    end)
    connections[connection] = true
    return connection
end

local function disconnect(connection)
    if connection then
        connection:Disconnect()
        connections[connection] = nil
    end
end



-- GUI Principal
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "CustomDevGui"
ScreenGui.ResetOnSpawn = false

local parentTarget = CoreGui
pcall(function()
	if gethui then
		parentTarget = gethui()
	end
end)
-- Cada nueva ejecución cierra limpiamente la anterior.
for _, previous in ipairs(parentTarget:GetChildren()) do
    if previous.Name == "CustomDevGui" and previous:IsA("ScreenGui") then
        local shutdown = previous:FindFirstChild("FrancoShutdown")
        if shutdown and shutdown:IsA("BindableFunction") then
            local ok = pcall(function() shutdown:Invoke() end)
            if not ok or previous.Parent then
                ScreenGui:Destroy()
                warn("No se pudo cerrar el panel anterior. Ciérralo con X y vuelve a ejecutar.")
                return
            end
        else
            ScreenGui:Destroy()
            warn("Cierra con el botón X la versión anterior del panel y vuelve a ejecutar esta versión.")
            return
        end
    end
end
local Shutdown = Instance.new("BindableFunction")
Shutdown.Name = "FrancoShutdown"
Shutdown.Parent = ScreenGui
ScreenGui.Parent = parentTarget
ScreenGui.DisplayOrder = 999999999

-- Aviso único: los mensajes nuevos reemplazan al anterior.
local notify
do
    local version = 0
    local toast = Instance.new("TextLabel")
    toast.Name = "PanelNotice"
    toast.AnchorPoint = Vector2.new(0.5, 1)
    toast.Position = UDim2.new(0.5, 0, 1, -18)
    toast.Size = UDim2.new(0.9, 0, 0, 54)
    toast.BackgroundColor3 = Color3.fromRGB(28, 19, 27)
    toast.BackgroundTransparency = 0.08
    toast.TextColor3 = Color3.fromRGB(248, 229, 234)
    toast.Font = Enum.Font.SourceSansBold
    toast.TextSize = 15
    toast.TextWrapped = true
    toast.Visible = false
    toast.ZIndex = 100
    toast.Parent = ScreenGui
    local limit = Instance.new("UISizeConstraint")
    limit.MaxSize = Vector2.new(360, 54)
    limit.Parent = toast
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = toast
    local padding = Instance.new("UIPadding")
    padding.PaddingLeft = UDim.new(0, 12)
    padding.PaddingRight = UDim.new(0, 12)
    padding.Parent = toast
    notify = function(message, failed)
        if closed then return end
        version = version + 1
        local current = version
        toast.Text = message
        toast.TextColor3 = failed and Color3.fromRGB(255, 150, 150) or Color3.fromRGB(248, 229, 234)
        toast.Visible = true
        task.delay(2.8, function()
            if not closed and current == version then toast.Visible = false end
        end)
    end
end

-- Marco Principal
local MainFrame = Instance.new("CanvasGroup")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 0, 0, 0)
MainFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(17, 12, 17)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.GroupTransparency = 1
MainFrame.Parent = ScreenGui

local MainStroke = Instance.new("UIStroke")
MainStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
MainStroke.Color = Color3.fromRGB(230, 42, 66)
MainStroke.Thickness = 2
MainStroke.Transparency = 0.2
MainStroke.Parent = MainFrame

local MainGradient = Instance.new("UIGradient")
MainGradient.Color = ColorSequence.new({
	ColorSequenceKeypoint.new(0, Color3.fromRGB(37, 16, 25)),
	ColorSequenceKeypoint.new(1, Color3.fromRGB(9, 8, 12))
})
MainGradient.Rotation = 45
MainGradient.Parent = MainFrame

local MainUICorner = Instance.new("UICorner")
MainUICorner.CornerRadius = UDim.new(0, 12)
MainUICorner.Parent = MainFrame

-- Header
local Header = Instance.new("Frame")
Header.Name = "Header"
Header.Size = UDim2.new(1, 0, 0, 52)
Header.BackgroundColor3 = Color3.fromRGB(25, 16, 23)
Header.BorderSizePixel = 0
Header.Parent = MainFrame

local HeaderUICorner = Instance.new("UICorner")
HeaderUICorner.CornerRadius = UDim.new(0, 12)
HeaderUICorner.Parent = Header

local ActiveDot = Instance.new("Frame")
ActiveDot.Name = "ActiveDot"
ActiveDot.Size = UDim2.new(0, 8, 0, 8)
ActiveDot.Position = UDim2.new(0, 12, 0.5, -4)
ActiveDot.BackgroundColor3 = Color3.fromRGB(50, 255, 100)
ActiveDot.BorderSizePixel = 0
ActiveDot.Parent = Header

local DotCorner = Instance.new("UICorner")
DotCorner.CornerRadius = UDim.new(1, 0)
DotCorner.Parent = ActiveDot

local DotStroke = Instance.new("UIStroke")
DotStroke.Color = Color3.fromRGB(50, 255, 100)
DotStroke.Thickness = 2
DotStroke.Transparency = 0.5
DotStroke.Parent = ActiveDot

task.spawn(function()
	while not closed and ActiveDot.Parent do
		TweenService:Create(ActiveDot, TweenInfo.new(0.6, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {BackgroundTransparency = 0.8}):Play()
		TweenService:Create(DotStroke, TweenInfo.new(0.6, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Transparency = 0.9}):Play()
		task.wait(0.6)
		if closed then return end
		TweenService:Create(ActiveDot, TweenInfo.new(0.6, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {BackgroundTransparency = 0}):Play()
		TweenService:Create(DotStroke, TweenInfo.new(0.6, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Transparency = 0.3}):Play()
		task.wait(0.6)
		if closed then return end
	end
end)

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -90, 1, 0)
Title.Position = UDim2.new(0, 26, 0, 0)
Title.BackgroundTransparency = 1
Title.Text = "<font color=\"#FF526F\">Franco</font>  <font color=\"#BD9EAA\">/ PANEL</font>"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.TextSize = 19
Title.Font = Enum.Font.GothamBold
Title.RichText = true
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = Header

local CloseBtn = Instance.new("TextButton")
CloseBtn.Size = UDim2.new(0, 26, 0, 26)
CloseBtn.Position = UDim2.new(1, -38, 0, 13)
CloseBtn.BackgroundColor3 = Color3.fromRGB(230, 50, 60)
CloseBtn.Text = "✕"
CloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseBtn.Font = Enum.Font.SourceSansBold
CloseBtn.TextSize = 13
CloseBtn.Parent = Header

local CloseCorner = Instance.new("UICorner")
CloseCorner.CornerRadius = UDim.new(0, 6)
CloseCorner.Parent = CloseBtn

local MinimizeBtn = Instance.new("TextButton")
MinimizeBtn.Size = UDim2.new(0, 26, 0, 26)
MinimizeBtn.Position = UDim2.new(1, -72, 0, 13)
MinimizeBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 65)
MinimizeBtn.Text = "─"
MinimizeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
MinimizeBtn.Font = Enum.Font.SourceSansBold
MinimizeBtn.TextSize = 12
MinimizeBtn.Parent = Header

local MinimizeCorner = Instance.new("UICorner")
MinimizeCorner.CornerRadius = UDim.new(0, 6)
MinimizeCorner.Parent = MinimizeBtn

--------------------------------------------------------------------------------
-- CÍRCULO FLOTANTE
--------------------------------------------------------------------------------
local FloatingCircle = Instance.new("TextButton")
FloatingCircle.Name = "FloatingCircle"
FloatingCircle.Size = UDim2.new(0, 52, 0, 52)
FloatingCircle.Position = UDim2.new(0.1, 0, 0.1, 0)
FloatingCircle.BackgroundColor3 = Color3.fromRGB(20, 15, 20)
FloatingCircle.Text = "F"
FloatingCircle.TextColor3 = Color3.fromRGB(255, 30, 30)
FloatingCircle.Font = Enum.Font.FredokaOne
FloatingCircle.TextSize = 28
FloatingCircle.Visible = false
FloatingCircle.Active = true
FloatingCircle.Parent = ScreenGui

local CircleCorner = Instance.new("UICorner")
CircleCorner.CornerRadius = UDim.new(1, 0)
CircleCorner.Parent = FloatingCircle

local CircleStroke = Instance.new("UIStroke")
CircleStroke.Color = Color3.fromRGB(255, 60, 0)
CircleStroke.Thickness = 2.5
CircleStroke.Transparency = 0.1
CircleStroke.Parent = FloatingCircle

local FlameGlow = Instance.new("UIStroke")
FlameGlow.ApplyStrokeMode = Enum.ApplyStrokeMode.Contextual
FlameGlow.Color = Color3.fromRGB(255, 0, 0)
FlameGlow.Thickness = 1.5
FlameGlow.Transparency = 0.3
FlameGlow.Parent = FloatingCircle

task.spawn(function()
	while not closed and FloatingCircle.Parent do
		TweenService:Create(CircleStroke, TweenInfo.new(0.4, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Color = Color3.fromRGB(255, 120, 0)}):Play()
		TweenService:Create(FloatingCircle, TweenInfo.new(0.4, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {TextColor3 = Color3.fromRGB(255, 60, 60)}):Play()
		task.wait(0.4)
		if closed then return end
		TweenService:Create(CircleStroke, TweenInfo.new(0.4, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Color = Color3.fromRGB(255, 0, 0)}):Play()
		TweenService:Create(FloatingCircle, TweenInfo.new(0.4, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {TextColor3 = Color3.fromRGB(255, 0, 0)}):Play()
		task.wait(0.4)
		if closed then return end
	end
end)

--------------------------------------------------------------------------------
-- TABS
--------------------------------------------------------------------------------
local TabSidebar = Instance.new("Frame")
TabSidebar.Name = "TabSidebar"
TabSidebar.Size = UDim2.new(0, 142, 1, -92)
TabSidebar.Position = UDim2.new(0, 12, 0, 64)
TabSidebar.BackgroundColor3 = Color3.fromRGB(22, 15, 22)
TabSidebar.BorderSizePixel = 0
TabSidebar.Parent = MainFrame

local SidebarCorner = Instance.new("UICorner")
SidebarCorner.CornerRadius = UDim.new(0, 8)
SidebarCorner.Parent = TabSidebar

local SidebarList = Instance.new("UIListLayout")
SidebarList.Parent = TabSidebar
SidebarList.SortOrder = Enum.SortOrder.LayoutOrder
SidebarList.Padding = UDim.new(0, 10)
local SidebarPadding = Instance.new("UIPadding")
SidebarPadding.PaddingTop = UDim.new(0, 8)
SidebarPadding.PaddingLeft = UDim.new(0, 6)
SidebarPadding.PaddingRight = UDim.new(0, 6)
SidebarPadding.Parent = TabSidebar

local Footer = Instance.new("TextLabel")
Footer.Size = UDim2.new(1, -24, 0, 20)
Footer.Position = UDim2.new(0, 12, 1, -24)
Footer.BackgroundTransparency = 1
Footer.Text = "FRANCO   ·   M: ocultar / mostrar menú"
Footer.TextColor3 = Color3.fromRGB(169, 133, 145)
Footer.Font = Enum.Font.Gotham
Footer.TextSize = 11
Footer.TextXAlignment = Enum.TextXAlignment.Left
Footer.Parent = MainFrame

local ContentContainer = Instance.new("Frame")
ContentContainer.Name = "ContentContainer"
ContentContainer.Size = UDim2.new(1, -178, 1, -92)
ContentContainer.Position = UDim2.new(0, 166, 0, 64)
ContentContainer.BackgroundTransparency = 1
ContentContainer.Parent = MainFrame

local tabs = {}
local function createTab(tabName, layoutOrder)
	local tabBtn = Instance.new("TextButton")
	tabBtn.Name = tabName .. "TabBtn"
	tabBtn.Size = UDim2.new(1, 0, 0, 44)
	tabBtn.BackgroundColor3 = Color3.fromRGB(29, 21, 29)
	local tabLabels = {
        ["Habilidades"] = "✦  Habilidades",
        ["criminales"] = "◆  Criminales",
        ["policie"] = "◈  Policía",
        ["TP jugadores"] = "◎  Jugadores",
    }
    tabBtn.Text = tabLabels[tabName] or tabName
    tabBtn.AutoButtonColor = false
	tabBtn.TextColor3 = Color3.fromRGB(160, 160, 175)
	tabBtn.Font = Enum.Font.GothamMedium
	tabBtn.TextSize = 13
	tabBtn.LayoutOrder = layoutOrder
	tabBtn.Parent = TabSidebar

	local btnCorner = Instance.new("UICorner")
	btnCorner.CornerRadius = UDim.new(0, 6)
	btnCorner.Parent = tabBtn

	local tabFrame = Instance.new("ScrollingFrame")
	tabFrame.Name = tabName .. "Page"
	tabFrame.Size = UDim2.new(1, 0, 1, 0)
	tabFrame.BackgroundTransparency = 1
	tabFrame.BorderSizePixel = 0
	tabFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
	tabFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
	tabFrame.ScrollBarThickness = 3
	tabFrame.ScrollBarImageColor3 = Color3.fromRGB(80, 80, 100)
	tabFrame.Visible = false
	tabFrame.Parent = ContentContainer

	local pageList = Instance.new("UIListLayout")
	pageList.Parent = tabFrame
	pageList.SortOrder = Enum.SortOrder.LayoutOrder
	pageList.Padding = UDim.new(0, 12)
    local pagePadding = Instance.new("UIPadding")
    pagePadding.PaddingTop = UDim.new(0, 3)
    pagePadding.PaddingBottom = UDim.new(0, 10)
    pagePadding.Parent = tabFrame

    local indicator = Instance.new("Frame")
    indicator.Size = UDim2.new(0, 3, 0.5, 0)
    indicator.Position = UDim2.new(0, 0, 0.25, 0)
    indicator.BackgroundColor3 = Color3.fromRGB(255, 151, 163)
    indicator.BorderSizePixel = 0
    indicator.Visible = false
    indicator.Parent = tabBtn
    local pageTween
    connect(tabBtn.MouseEnter, function()
        if not tabFrame.Visible then
            TweenService:Create(tabBtn, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(64, 30, 42)}):Play()
        end
    end)
    connect(tabBtn.MouseLeave, function()
        if not tabFrame.Visible then
            TweenService:Create(tabBtn, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(29, 21, 29)}):Play()
        end
    end)

	connect(tabBtn.MouseButton1Click, function()
		for name, data in pairs(tabs) do
			data.Frame.Visible = false
            data.Indicator.Visible = false
			TweenService:Create(data.Button, TweenInfo.new(0.2), {
				BackgroundColor3 = Color3.fromRGB(29, 21, 29),
				TextColor3 = Color3.fromRGB(160, 160, 175)
			}):Play()
		end
        if pageTween then pageTween:Cancel() end
        tabFrame.Position = UDim2.fromOffset(8, 0)
        tabFrame.Visible = true
        indicator.Visible = true
        pageTween = TweenService:Create(tabFrame, TweenInfo.new(0.2, Enum.EasingStyle.Quad), {Position = UDim2.fromOffset(0, 0)})
        pageTween:Play()
		TweenService:Create(tabBtn, TweenInfo.new(0.2), {
			BackgroundColor3 = Color3.fromRGB(174, 31, 54),
			TextColor3 = Color3.fromRGB(255, 255, 255)
		}):Play()
	end)

	tabs[tabName] = { Button = tabBtn, Frame = tabFrame, Indicator = indicator }
	return tabFrame
end

local HabilidadesPage = createTab("Habilidades", 1)
local CriminalesPage = createTab("criminales", 2)
local PoliciePage = createTab("policie", 3)
local TPJugadoresPage = createTab("TP jugadores", 4)

tabs["Habilidades"].Frame.Visible = true
tabs["Habilidades"].Indicator.Visible = true
tabs["Habilidades"].Button.BackgroundColor3 = Color3.fromRGB(174, 31, 54)
tabs["Habilidades"].Button.TextColor3 = Color3.fromRGB(255, 255, 255)

--------------------------------------------------------------------------------
-- HABILIDADES
--------------------------------------------------------------------------------
local InfoBox = Instance.new("Frame")
InfoBox.Size = UDim2.new(1, -10, 0, 52)
InfoBox.BackgroundColor3 = Color3.fromRGB(28, 19, 27)
InfoBox.BorderSizePixel = 0
InfoBox.Parent = HabilidadesPage

local InfoCorner = Instance.new("UICorner")
InfoCorner.CornerRadius = UDim.new(0, 6)
InfoCorner.Parent = InfoBox

local InfoStroke = Instance.new("UIStroke")
InfoStroke.Color = Color3.fromRGB(188, 44, 67)
InfoStroke.Thickness = 1
InfoStroke.Transparency = 0.5
InfoStroke.Parent = InfoBox

local InfoText = Instance.new("TextLabel")
InfoText.Size = UDim2.new(1, -12, 1, -8)
InfoText.Position = UDim2.new(0, 6, 0, 4)
InfoText.BackgroundTransparency = 1
InfoText.Text = "FRANCO  /  HABILIDADES\nConfigura tus controles desde este panel."
InfoText.TextColor3 = Color3.fromRGB(200, 210, 230)
InfoText.Font = Enum.Font.SourceSansItalic
InfoText.TextSize = 13
InfoText.TextWrapped = true
InfoText.TextYAlignment = Enum.TextYAlignment.Center
InfoText.Parent = InfoBox

local VisulyBtn = Instance.new("TextButton")
VisulyBtn.Size = UDim2.new(1, -10, 0, 40)
VisulyBtn.BackgroundColor3 = Color3.fromRGB(36, 24, 33)
VisulyBtn.Text = "Visuly: OFF  [Tecla V]"
VisulyBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
VisulyBtn.Font = Enum.Font.SourceSans
VisulyBtn.TextSize = 15
VisulyBtn.Parent = HabilidadesPage

local VisulyCorner = Instance.new("UICorner")
VisulyCorner.CornerRadius = UDim.new(0, 6)
VisulyCorner.Parent = VisulyBtn

local JumpBtn = Instance.new("TextButton")
JumpBtn.Size = UDim2.new(1, -10, 0, 40)
JumpBtn.BackgroundColor3 = Color3.fromRGB(36, 24, 33)
JumpBtn.Text = "Súper Salto: OFF  [Tecla J]"
JumpBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
JumpBtn.Font = Enum.Font.SourceSans
JumpBtn.TextSize = 15
JumpBtn.Parent = HabilidadesPage

local JumpCorner = Instance.new("UICorner")
JumpCorner.CornerRadius = UDim.new(0, 10)
JumpCorner.Parent = JumpBtn

local TpForwardBtn = Instance.new("TextButton")
TpForwardBtn.Size = UDim2.new(1, -10, 0, 40)
TpForwardBtn.BackgroundColor3 = Color3.fromRGB(36, 24, 33)
TpForwardBtn.Text = "Teleport 5 met..  [Tecla T]"
TpForwardBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
TpForwardBtn.Font = Enum.Font.SourceSans
TpForwardBtn.TextSize = 15
TpForwardBtn.Parent = HabilidadesPage

local TpForwardCorner = Instance.new("UICorner")
TpForwardCorner.CornerRadius = UDim.new(0, 6)
TpForwardCorner.Parent = TpForwardBtn

--------------------------------------------------------------------------------
-- CRIMINALES
--------------------------------------------------------------------------------
local CrimSubTitle = Instance.new("TextLabel")
CrimSubTitle.Size = UDim2.new(1, -10, 0, 15)
CrimSubTitle.BackgroundTransparency = 1
CrimSubTitle.Text = "presiona para ir a una base criminal"
CrimSubTitle.TextColor3 = Color3.fromRGB(150, 150, 165)
CrimSubTitle.Font = Enum.Font.SourceSansItalic
CrimSubTitle.TextSize = 12
CrimSubTitle.TextXAlignment = Enum.TextXAlignment.Left
CrimSubTitle.Parent = CriminalesPage

local CrimTpBtn = Instance.new("TextButton")
CrimTpBtn.Size = UDim2.new(1, -10, 0, 40)
CrimTpBtn.BackgroundColor3 = Color3.fromRGB(36, 24, 33)
CrimTpBtn.Text = "base.crim  [Tecla B]"
CrimTpBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
CrimTpBtn.Font = Enum.Font.SourceSans
CrimTpBtn.TextSize = 15
CrimTpBtn.Parent = CriminalesPage

local CrimTpCorner = Instance.new("UICorner")
CrimTpCorner.CornerRadius = UDim.new(0, 6)
CrimTpCorner.Parent = CrimTpBtn

local SeldaTpBtn = Instance.new("TextButton")
SeldaTpBtn.Size = UDim2.new(1, -10, 0, 40)
SeldaTpBtn.BackgroundColor3 = Color3.fromRGB(36, 24, 33)
SeldaTpBtn.Text = "selda  [Tecla Z]"
SeldaTpBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
SeldaTpBtn.Font = Enum.Font.SourceSans
SeldaTpBtn.TextSize = 15
SeldaTpBtn.Parent = CriminalesPage

local SeldaTpCorner = Instance.new("UICorner")
SeldaTpCorner.CornerRadius = UDim.new(0, 6)
SeldaTpCorner.Parent = SeldaTpBtn

--------------------------------------------------------------------------------
-- POLICIE
--------------------------------------------------------------------------------
local PlcSubTitle = Instance.new("TextLabel")
PlcSubTitle.Size = UDim2.new(1, -10, 0, 15)
PlcSubTitle.BackgroundTransparency = 1
PlcSubTitle.Text = "presiona para ir a la base policial"
PlcSubTitle.TextColor3 = Color3.fromRGB(150, 150, 165)
PlcSubTitle.Font = Enum.Font.SourceSansItalic
PlcSubTitle.TextSize = 12
PlcSubTitle.TextXAlignment = Enum.TextXAlignment.Left
PlcSubTitle.Parent = PoliciePage

local BasePlcBtn = Instance.new("TextButton")
BasePlcBtn.Size = UDim2.new(1, -10, 0, 40)
BasePlcBtn.BackgroundColor3 = Color3.fromRGB(36, 24, 33)
BasePlcBtn.Text = "Base PLC.  [Tecla P]"
BasePlcBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
BasePlcBtn.Font = Enum.Font.SourceSans
BasePlcBtn.TextSize = 15
BasePlcBtn.Parent = PoliciePage

local BasePlcCorner = Instance.new("UICorner")
BasePlcCorner.CornerRadius = UDim.new(0, 6)
BasePlcCorner.Parent = BasePlcBtn

--------------------------------------------------------------------------------
-- BOTÓN AIM MÓVIL
--------------------------------------------------------------------------------
local MobileAimBtn = Instance.new("TextButton")
MobileAimBtn.Name = "MobileAimBtn"
MobileAimBtn.Size = UDim2.new(0, 70, 0, 70)
MobileAimBtn.Position = UDim2.new(0.8, 0, 0.5, 0)
MobileAimBtn.BackgroundColor3 = Color3.fromRGB(220, 50, 50)
MobileAimBtn.BackgroundTransparency = 0.2
MobileAimBtn.Text = "AIM Movil"
MobileAimBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
MobileAimBtn.Font = Enum.Font.SourceSansBold
MobileAimBtn.TextSize = 13
MobileAimBtn.Active = true
MobileAimBtn.Parent = ScreenGui

local MobileAimCorner = Instance.new("UICorner")
MobileAimCorner.CornerRadius = UDim.new(1, 0)
MobileAimCorner.Parent = MobileAimBtn

if not UserInputService.TouchEnabled then
	MobileAimBtn.Visible = false
end

--------------------------------------------------------------------------------
-- APERTURA
--------------------------------------------------------------------------------
local targetSize = UDim2.new(0, 520, 0, 390)
local targetPos = UDim2.new(0.5, -260, 0.5, -195)

TweenService:Create(MainFrame, TweenInfo.new(0.65, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = targetSize, Position = targetPos}):Play()
TweenService:Create(MainFrame, TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {GroupTransparency = 0}):Play()

--------------------------------------------------------------------------------
-- DRAG
--------------------------------------------------------------------------------
local function enableDrag(frame)
	local dragging = false
	local dragInput, dragStart, startPosition

	local function update(input)
		local delta = input.Position - dragStart
		frame.Position = UDim2.new(
			startPosition.X.Scale,
			startPosition.X.Offset + delta.X,
			startPosition.Y.Scale,
			startPosition.Y.Offset + delta.Y
		)
	end

	connect(frame.InputBegan, function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			dragStart = input.Position
			startPosition = frame.Position
			local endConnection
			endConnection = connect(input.Changed, function()
				if input.UserInputState == Enum.UserInputState.End then
					dragging = false
					disconnect(endConnection)
				end
			end)
		end
	end)

	connect(frame.InputChanged, function(input)
		if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
			dragInput = input
		end
	end)

	connect(UserInputService.InputChanged, function(input)
		if input == dragInput and dragging then
			update(input)
		end
	end)
end

-- El botón AIM conserva exactamente su arrastre anterior.
enableDrag(MobileAimBtn)

local function clampToScreen(frame)
    local available = ScreenGui.AbsoluteSize
    if available.X <= 0 or available.Y <= 0 then return end
    local origin = ScreenGui.AbsolutePosition
    local position = frame.AbsolutePosition - origin
    local size = frame.AbsoluteSize
    frame.Position = UDim2.fromOffset(
        math.clamp(position.X, 0, math.max(0, available.X - size.X)),
        math.clamp(position.Y, 0, math.max(0, available.Y - size.Y))
    )
end

-- Un solo dedo controla cada arrastre; un desplazamiento no cuenta como toque.
local function enablePanelDrag(handle, frame)
    local activeInput, startPointer, startPosition
    local moved = false
    handle.Active = true
    connect(handle.InputBegan, function(input)
        if activeInput then return end
        if input.UserInputType ~= Enum.UserInputType.Touch
            and input.UserInputType ~= Enum.UserInputType.MouseButton1 then return end
        activeInput = input
        startPointer = input.Position
        startPosition = frame.AbsolutePosition - ScreenGui.AbsolutePosition
        moved = false
    end)
    connect(UserInputService.InputChanged, function(input)
        if not activeInput then return end
        local isTouch = activeInput.UserInputType == Enum.UserInputType.Touch
        if (isTouch and input ~= activeInput)
            or (not isTouch and input.UserInputType ~= Enum.UserInputType.MouseMovement) then return end
        local delta = input.Position - startPointer
        if delta.Magnitude >= 8 then moved = true end
        if moved then
            frame.Position = UDim2.fromOffset(startPosition.X + delta.X, startPosition.Y + delta.Y)
            clampToScreen(frame)
        end
    end)
    connect(UserInputService.InputEnded, function(input)
        if input == activeInput then activeInput = nil end
    end)
    return function() return moved end
end

-- Arrastrar desde el título deja libres los controles y el desplazamiento de listas.
enablePanelDrag(Title, MainFrame)
local floatingWasDragged = enablePanelDrag(FloatingCircle, FloatingCircle)

local PanelScale = Instance.new("UIScale")
PanelScale.Parent = MainFrame
local layoutReady = false
local function fitPanel()
    local available = ScreenGui.AbsoluteSize
    if available.X <= 0 or available.Y <= 0 then return end
    PanelScale.Scale = math.min(1, math.max(0.1, (available.X - 32) / 520),
        math.max(0.1, (available.Y - 32) / 390))
    if layoutReady then
        clampToScreen(MainFrame)
        clampToScreen(FloatingCircle)
    end
end
connect(ScreenGui:GetPropertyChangedSignal("AbsoluteSize"), fitPanel)
fitPanel()
task.delay(0.7, function()
    if closed then return end
    layoutReady = true
    local available = ScreenGui.AbsoluteSize
    MainFrame.Position = UDim2.fromOffset(
        (available.X - 520 * PanelScale.Scale) / 2,
        (available.Y - 390 * PanelScale.Scale) / 2)
    fitPanel()
end)

--------------------------------------------------------------------------------
-- MINIMIZAR
--------------------------------------------------------------------------------
local menuOpen = true
local menuAnimation, menuRevision = nil, 0
local function showMenu(visible, showCircle)
    menuOpen = visible
    menuRevision = menuRevision + 1
    local revision = menuRevision
    if menuAnimation then menuAnimation:Cancel() end
    FloatingCircle.Visible = not visible and showCircle
    if visible then MainFrame.Visible = true end
    menuAnimation = TweenService:Create(MainFrame, TweenInfo.new(0.16, Enum.EasingStyle.Quad), {
        GroupTransparency = visible and 0 or 1,
    })
    menuAnimation:Play()
    if not visible then
        task.delay(0.17, function()
            if not closed and revision == menuRevision then MainFrame.Visible = false end
        end)
    end
end
local function toggleMinimize()
    showMenu(not menuOpen, true)
end
local function toggleMenuKey()
    showMenu(not menuOpen, false)
end

connect(MinimizeBtn.MouseButton1Click, toggleMinimize)
connect(FloatingCircle.Activated, function()
    if not floatingWasDragged() then toggleMinimize() end
end)

--------------------------------------------------------------------------------
-- Rayos y partículas 2D: objetos reutilizados, sin imágenes ni descargas.
do
    local layer = Instance.new("Frame")
    layer.Name = "FrancoElectricBorder"
    layer.BackgroundTransparency = 1
    layer.BorderSizePixel = 0
    layer.ClipsDescendants = false
    layer.Active = false
    layer.ZIndex = 0
    layer.Parent = ScreenGui
    local bolts, sparks = {}, {}
    local clockTime, accumulated = 0, 0

    local function edgePoint(side, along, outward, width, height)
        if side == 1 then return Vector2.new(along * width, -outward) end
        if side == 2 then return Vector2.new(width + outward, along * height) end
        if side == 3 then return Vector2.new(along * width, height + outward) end
        return Vector2.new(-outward, along * height)
    end
    for i = 1, 8 do
        local bolt = { Side = (i - 1) % 4 + 1, Along = 0.18 + math.random() * 0.64, Segments = {}, Shadows = {} }
        for j = 1, 3 do
            local line = Instance.new("Frame")
            line.AnchorPoint = Vector2.new(0.5, 0.5)
            line.BackgroundColor3 = j == 1 and Color3.fromRGB(255, 137, 152) or Color3.fromRGB(210, 27, 51)
            line.BorderSizePixel = 0
            line.ZIndex = 0
            line.Parent = layer
            bolt.Segments[j] = line
            local shadow = Instance.new("Frame")
            shadow.AnchorPoint = Vector2.new(0.5, 0.5)
            shadow.BackgroundColor3 = Color3.fromRGB(5, 3, 7)
            shadow.BorderSizePixel = 0
            shadow.ZIndex = 0
            shadow.Parent = layer
            line.ZIndex = 1
            bolt.Shadows[j] = shadow
        end
        bolts[i] = bolt
    end
    for i = 1, 12 do
        local dot = Instance.new("Frame")
        dot.AnchorPoint = Vector2.new(0.5, 0.5)
        dot.BorderSizePixel = 0
        dot.BackgroundColor3 = Color3.fromRGB(241, 61, 85)
        dot.ZIndex = 0
        dot.Parent = layer
        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(1, 0)
        corner.Parent = dot
        sparks[i] = { Dot = dot, Side = (i - 1) % 4 + 1, Along = math.random(), Phase = i / 12 }
    end
    connect(RunService.Heartbeat, function(dt)
        layer.Visible = MainFrame.Visible and MainFrame.GroupTransparency < 0.95
        if not layer.Visible then return end
        -- Sigue el arrastre sin crear instancias por fotograma.
        layer.Position = UDim2.fromOffset(
            MainFrame.AbsolutePosition.X - ScreenGui.AbsolutePosition.X,
            MainFrame.AbsolutePosition.Y - ScreenGui.AbsolutePosition.Y)
        layer.Size = UDim2.fromOffset(MainFrame.AbsoluteSize.X, MainFrame.AbsoluteSize.Y)
        accumulated = accumulated + dt
        clockTime = clockTime + dt
        if accumulated < 1 / 24 then return end
        accumulated = 0
        local width, height = MainFrame.AbsoluteSize.X, MainFrame.AbsoluteSize.Y
        local scale = PanelScale.Scale
        local opacity = MainFrame.GroupTransparency
        for i, bolt in ipairs(bolts) do
            local pulse = (clockTime * 0.85 + i * 0.137) % 1
            local visible = pulse < 0.28
            local previous = edgePoint(bolt.Side, bolt.Along, 1, width, height)
            for j, line in ipairs(bolt.Segments) do
                line.Visible = visible
                bolt.Shadows[j].Visible = visible
                if visible then
                    local jitter = math.sin(clockTime * 16 + i * 9 + j * 3) * 0.014
                    local point = edgePoint(bolt.Side, bolt.Along + jitter, j * 7 * scale, width, height)
                    local delta = point - previous
                    local middle = (point + previous) / 2
                    line.Position = UDim2.fromOffset(middle.X, middle.Y)
                    line.Size = UDim2.fromOffset(delta.Magnitude, math.max(1, 1.5 * scale))
                    line.Rotation = math.deg(math.atan2(delta.Y, delta.X))
                    line.BackgroundTransparency = math.max(opacity, 0.22 + pulse * 1.8)
                    local shadow = bolt.Shadows[j]
                    shadow.Position = line.Position
                    shadow.Size = UDim2.fromOffset(delta.Magnitude + 3, math.max(4, 5 * scale))
                    shadow.Rotation = line.Rotation
                    shadow.BackgroundTransparency = math.max(opacity, 0.18 + pulse)
                    previous = point
                end
            end
        end
        for _, spark in ipairs(sparks) do
            local progress = (clockTime * 0.55 + spark.Phase) % 1
            local point = edgePoint(spark.Side, spark.Along, (3 + progress * 24) * scale, width, height)
            spark.Dot.Position = UDim2.fromOffset(point.X, point.Y)
            spark.Dot.Size = UDim2.fromOffset(3 * scale, 3 * scale)
            spark.Dot.BackgroundTransparency = math.max(opacity, 0.25 + progress * 0.75)
        end
    end)
end


-- ESP (contorno de enemigos)
--------------------------------------------------------------------------------
local visulyEnabled = false

local function isEnemy(player)
	if player == LocalPlayer then return false end
	if not player.Character then return false end

	if LocalPlayer.Team and player.Team then
		return LocalPlayer.Team ~= player.Team
	end

	if LocalPlayer.TeamColor and player.TeamColor then
		return LocalPlayer.TeamColor ~= player.TeamColor
	end

	return false
end



local function applyHighlight(player)
	if not isEnemy(player) or not player.Character then return end

	-- Outline rojo
	if not player.Character:FindFirstChild("VisulyHighlight") then
		local highlight = Instance.new("Highlight")
		highlight.Name = "VisulyHighlight"
		highlight.OutlineColor = Color3.fromRGB(255, 40, 40)
		highlight.OutlineTransparency = 0
		highlight.FillTransparency = 1
		highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
		highlight.Adornee = player.Character
		highlight.Parent = player.Character
	end

end

local function removePlayerHighlight(player)
    local character = player.Character
    if not character then return end
    local highlight = character:FindFirstChild("VisulyHighlight")
    if highlight then highlight:Destroy() end
end

local function removeHighlights()
    for _, player in ipairs(Players:GetPlayers()) do
        removePlayerHighlight(player)
    end
end
local function updateHighlights()
	removeHighlights()
	if visulyEnabled then
		for _, player in ipairs(Players:GetPlayers()) do
			applyHighlight(player)
		end
	end
end

local function toggleVisuly()
	visulyEnabled = not visulyEnabled
    notify(visulyEnabled and "Visuly activado" or "Visuly desactivado")
	if visulyEnabled then
		VisulyBtn.Text = "Visuly: ON  [Tecla V]"
		VisulyBtn.BackgroundColor3 = Color3.fromRGB(146, 27, 47)
		updateHighlights()
	else
		VisulyBtn.Text = "Visuly: OFF  [Tecla V]"
		VisulyBtn.BackgroundColor3 = Color3.fromRGB(36, 24, 33)
		removeHighlights()
	end
end

connect(VisulyBtn.MouseButton1Click, toggleVisuly)

-- Guarda y restaura la configuración original de cada personaje.
local jumpEnabled = false
local superJumpHeight = 50
local savedJump = nil

local function restoreJump()
    if savedJump then
        local humanoid = savedJump.Humanoid
        if humanoid.Parent then
            humanoid.JumpHeight = savedJump.JumpHeight
            humanoid.JumpPower = savedJump.JumpPower
            humanoid.UseJumpPower = savedJump.UseJumpPower
        end
        savedJump = nil
    end
end

local function applySuperJump(humanoid)
    if closed or not jumpEnabled then return end
    if not savedJump or savedJump.Humanoid ~= humanoid then
        restoreJump()
        savedJump = {
            Humanoid = humanoid,
            JumpHeight = humanoid.JumpHeight,
            JumpPower = humanoid.JumpPower,
            UseJumpPower = humanoid.UseJumpPower,
        }
    end
    humanoid.UseJumpPower = false
    humanoid.JumpHeight = superJumpHeight
end

local function disableSuperJump()
    jumpEnabled = false
    restoreJump()
    JumpBtn.Text = "Súper Salto: OFF  [Tecla J]"
    JumpBtn.BackgroundColor3 = Color3.fromRGB(36, 24, 33)
end

local function toggleSuperJump()
    if jumpEnabled then
        disableSuperJump()
        notify("Súper salto desactivado")
        return
    end
    jumpEnabled = true
    notify("Súper salto activado")
    JumpBtn.Text = "Súper Salto: ON  [Tecla J]"
    JumpBtn.BackgroundColor3 = Color3.fromRGB(146, 27, 47)
    local character = LocalPlayer.Character
    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
    if humanoid then applySuperJump(humanoid) end
end

connect(JumpBtn.MouseButton1Click, toggleSuperJump)

-- VELOCIDAD: el ajuste empieza apagado y se mantiene al reaparecer.
local attachSpeed, releaseSpeed, disableSpeed
do
    local minimum, maximum = 16, 100
    local selected = 32
    local enabled = false
    local humanoid, originalSpeed, speedConnection
    local sliderInput
    local wasScrolling

    local card = Instance.new("Frame")
    card.Name = "SpeedControl"
    card.Size = UDim2.new(1, -10, 0, 130)
    card.LayoutOrder = 10
    card.BackgroundColor3 = Color3.fromRGB(28, 19, 27)
    card.BorderSizePixel = 0
    card.Parent = HabilidadesPage
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = card

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -20, 0, 25)
    label.Position = UDim2.fromOffset(10, 6)
    label.BackgroundTransparency = 1
    label.TextColor3 = Color3.fromRGB(255, 255, 255)
    label.Font = Enum.Font.SourceSansBold
    label.TextSize = 15
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = card

    local slider = Instance.new("TextButton")
    slider.Name = "SpeedSlider"
    slider.Size = UDim2.new(1, -32, 0, 36)
    slider.Position = UDim2.fromOffset(16, 32)
    slider.BackgroundTransparency = 1
    slider.Text = ""
    slider.AutoButtonColor = false
    slider.Parent = card

    local track = Instance.new("Frame")
    track.Size = UDim2.new(1, 0, 0, 6)
    track.Position = UDim2.new(0, 0, 0.5, -3)
    track.BackgroundColor3 = Color3.fromRGB(55, 55, 70)
    track.BorderSizePixel = 0
    track.Parent = slider
    local trackCorner = Instance.new("UICorner")
    trackCorner.CornerRadius = UDim.new(1, 0)
    trackCorner.Parent = track

    local fill = Instance.new("Frame")
    fill.BackgroundColor3 = Color3.fromRGB(174, 31, 54)
    fill.BorderSizePixel = 0
    fill.Parent = track
    local fillCorner = Instance.new("UICorner")
    fillCorner.CornerRadius = UDim.new(1, 0)
    fillCorner.Parent = fill

    local knob = Instance.new("Frame")
    knob.Size = UDim2.fromOffset(18, 18)
    knob.AnchorPoint = Vector2.new(0.5, 0.5)
    knob.BackgroundColor3 = Color3.fromRGB(248, 229, 234)
    knob.BorderSizePixel = 0
    knob.Parent = track
    local knobCorner = Instance.new("UICorner")
    knobCorner.CornerRadius = UDim.new(1, 0)
    knobCorner.Parent = knob

    local toggle = Instance.new("TextButton")
    toggle.Size = UDim2.new(0.5, -15, 0, 34)
    toggle.Position = UDim2.fromOffset(10, 76)
    toggle.TextColor3 = Color3.fromRGB(255, 255, 255)
    toggle.Font = Enum.Font.SourceSansBold
    toggle.TextSize = 14
    toggle.Parent = card
    local toggleCorner = Instance.new("UICorner")
    toggleCorner.CornerRadius = UDim.new(0, 6)
    toggleCorner.Parent = toggle

    local reset = Instance.new("TextButton")
    reset.Size = UDim2.new(0.5, -15, 0, 34)
    reset.Position = UDim2.new(0.5, 5, 0, 76)
    reset.BackgroundColor3 = Color3.fromRGB(50, 50, 65)
    reset.Text = "Restaurar normal"
    reset.TextColor3 = Color3.fromRGB(255, 255, 255)
    reset.Font = Enum.Font.SourceSans
    reset.TextSize = 14
    reset.Parent = card
    local resetCorner = Instance.new("UICorner")
    resetCorner.CornerRadius = UDim.new(0, 6)
    resetCorner.Parent = reset

    local function refresh()
        local ratio = (selected - minimum) / (maximum - minimum)
        fill.Size = UDim2.new(ratio, 0, 1, 0)
        knob.Position = UDim2.new(ratio, 0, 0.5, 0)
        label.Text = "Velocidad: " .. selected .. (enabled and "  · ON" or "  · OFF")
        toggle.Text = enabled and "Desactivar" or "Activar"
        toggle.BackgroundColor3 = enabled and Color3.fromRGB(146, 27, 47) or Color3.fromRGB(174, 31, 54)
    end

    local function enforceSpeed()
        if closed or not enabled or not humanoid or not humanoid.Parent then return end
        if humanoid.WalkSpeed ~= selected then humanoid.WalkSpeed = selected end
    end

    releaseSpeed = function()
        disconnect(speedConnection)
        speedConnection = nil
        if humanoid and humanoid.Parent and originalSpeed ~= nil then
            humanoid.WalkSpeed = originalSpeed
        end
        humanoid, originalSpeed = nil, nil
    end

    attachSpeed = function(newHumanoid)
        if closed or not enabled then return end
        if humanoid == newHumanoid and speedConnection then return end
        releaseSpeed()
        humanoid = newHumanoid
        originalSpeed = humanoid.WalkSpeed
        enforceSpeed()
        -- Reaplica después de cambios del sistema de correr (Shift).
        -- No escucha sus propias escrituras: evita bucles de eventos.
        speedConnection = connect(RunService.Heartbeat, enforceSpeed)
    end

    disableSpeed = function()
        enabled = false
        releaseSpeed()
        refresh()
        notify("Velocidad normal restaurada")
    end

    connect(toggle.Activated, function()
        if enabled then disableSpeed() return end
        enabled = true
        notify("Velocidad activada: " .. selected)
        local character = LocalPlayer.Character
        local current = character and character:FindFirstChildOfClass("Humanoid")
        if current then attachSpeed(current) end
        refresh()
    end)
    connect(reset.Activated, disableSpeed)

    local function setFromPointer(position)
        if slider.AbsoluteSize.X <= 0 then return end
        local ratio = math.clamp((position.X - slider.AbsolutePosition.X) / slider.AbsoluteSize.X, 0, 1)
        selected = math.floor(minimum + ratio * (maximum - minimum) + 0.5)
        enforceSpeed()
        refresh()
    end

    local function endSlider()
        if not sliderInput then return end
        sliderInput = nil
        HabilidadesPage.ScrollingEnabled = wasScrolling
        notify("Velocidad elegida: " .. selected .. (enabled and "" or " · Pulsa Activar"))
    end
    connect(slider.InputBegan, function(input)
        if sliderInput then return end
        if input.UserInputType ~= Enum.UserInputType.Touch
            and input.UserInputType ~= Enum.UserInputType.MouseButton1 then return end
        sliderInput = input
        wasScrolling = HabilidadesPage.ScrollingEnabled
        HabilidadesPage.ScrollingEnabled = false
        setFromPointer(input.Position)
    end)
    connect(UserInputService.InputChanged, function(input)
        if not sliderInput then return end
        local touch = sliderInput.UserInputType == Enum.UserInputType.Touch
        if (touch and input == sliderInput)
            or (not touch and input.UserInputType == Enum.UserInputType.MouseMovement) then
            setFromPointer(input.Position)
        end
    end)
    connect(UserInputService.InputEnded, function(input)
        if input == sliderInput then endSlider() end
    end)
    connect(UserInputService.WindowFocusReleased, endSlider)
    connect(MainFrame:GetPropertyChangedSignal("Visible"), endSlider)
    connect(HabilidadesPage:GetPropertyChangedSignal("Visible"), endSlider)
    refresh()
end


local function onCharacterAdded(player, character)
    local humanoid = character:WaitForChild("Humanoid", 10)
    if closed or player.Character ~= character or player.Parent ~= Players then return end
    if player == LocalPlayer and humanoid then
        applySuperJump(humanoid)
        attachSpeed(humanoid)
    end
    local head = character:WaitForChild("Head", 10)
    if closed or player.Character ~= character or player.Parent ~= Players then return end
    if visulyEnabled and head then applyHighlight(player) end
end

local function onTeamChanged(player)
    if visulyEnabled then
        if player == LocalPlayer then
            updateHighlights()
        else
            removePlayerHighlight(player)
            applyHighlight(player)
        end
    end
    if updatePlayerList then updatePlayerList() end
end

local function registerPlayer(player)
    if playerConnections[player] then return end
    playerConnections[player] = {
        connect(player.CharacterAdded, function(character)
            onCharacterAdded(player, character)
        end),
        connect(player.CharacterRemoving, function()
            removePlayerHighlight(player)
            if player == LocalPlayer then
                restoreJump()
                releaseSpeed()
            end
        end),
        connect(player:GetPropertyChangedSignal("Team"), function()
            onTeamChanged(player)
        end),
        connect(player:GetPropertyChangedSignal("TeamColor"), function()
            onTeamChanged(player)
        end),
    }
    if player.Character then
        task.spawn(onCharacterAdded, player, player.Character)
    end
end

for _, player in ipairs(Players:GetPlayers()) do registerPlayer(player) end
connect(Players.PlayerAdded, registerPlayer)
connect(Players.PlayerRemoving, function(player)
    removePlayerHighlight(player)
    for _, connection in ipairs(playerConnections[player] or {}) do
        disconnect(connection)
    end
    playerConnections[player] = nil
end)
local function executeTeleportForward()
	local character = LocalPlayer.Character
	if character and character:FindFirstChild("HumanoidRootPart") then
		local hrp = character.HumanoidRootPart
		hrp.CFrame = hrp.CFrame + (hrp.CFrame.LookVector * 16.4)
        notify("Personaje movido hacia delante")
    else
        notify("Tu personaje todavía no está disponible", true)
	end
end

connect(TpForwardBtn.MouseButton1Click, executeTeleportForward)

--------------------------------------------------------------------------------
-- TELEPORTS
--------------------------------------------------------------------------------
local function teleportTo(targetObject)
    local character = LocalPlayer.Character
    local root = character and character:FindFirstChild("HumanoidRootPart")
    if not root or not targetObject then
        notify("No se pudo ir: personaje o destino no disponible", true)
        return
    end
    local ok, moved = pcall(function()
        if targetObject:IsA("BasePart") then
            root.CFrame = targetObject.CFrame * CFrame.new(0, 3, 0)
        elseif targetObject:IsA("Model") then
            root.CFrame = targetObject:GetPivot() * CFrame.new(0, 3, 0)
        else
            return false
        end
        return true
    end)
    if ok and moved then
        notify("Personaje movido al destino")
    else
        notify("No se pudo ir a ese destino", true)
    end
end

local function tpBaseCrim()
    local base = Workspace:FindFirstChild("Criminals Spawn")
    teleportTo(base and base:FindFirstChild("SpawnLocation"))
end

local function tpSelda()
    local prison = Workspace:FindFirstChild("Prison_spawn")
    local courtyard = prison and prison:FindFirstChild("Courtyard")
    teleportTo(courtyard and courtyard:GetChildren()[2])
end

local function tpBasePlc()
    teleportTo(Workspace:FindFirstChild("spawn"))
end

connect(CrimTpBtn.MouseButton1Click, tpBaseCrim)
connect(SeldaTpBtn.MouseButton1Click, tpSelda)
connect(BasePlcBtn.MouseButton1Click, tpBasePlc)

--------------------------------------------------------------------------------
-- KEYBINDS
--------------------------------------------------------------------------------
connect(UserInputService.InputBegan, function(input, gameProcessed)
	if gameProcessed or UserInputService:GetFocusedTextBox() then return end

	if input.KeyCode == Enum.KeyCode.V then
		toggleVisuly()
	elseif input.KeyCode == Enum.KeyCode.J then
		toggleSuperJump()
	elseif input.KeyCode == Enum.KeyCode.T then
		executeTeleportForward()
	elseif input.KeyCode == Enum.KeyCode.B then
		tpBaseCrim()
	elseif input.KeyCode == Enum.KeyCode.Z then
		tpSelda()
	elseif input.KeyCode == Enum.KeyCode.P then
		tpBasePlc()
	elseif input.KeyCode == Enum.KeyCode.M then
        toggleMenuKey()
	elseif input.KeyCode == Enum.KeyCode.X then
		toggleMinimize()
	end
end)

--------------------------------------------------------------------------------
-- TP JUGADORES
--------------------------------------------------------------------------------
local PlayerSearch = Instance.new("TextBox")
PlayerSearch.Name = "PlayerSearch"
PlayerSearch.Size = UDim2.new(1, -10, 0, 36)
PlayerSearch.LayoutOrder = -2
PlayerSearch.BackgroundColor3 = Color3.fromRGB(36, 24, 33)
PlayerSearch.BorderSizePixel = 0
PlayerSearch.PlaceholderText = "Buscar nombre o @usuario..."
PlayerSearch.PlaceholderColor3 = Color3.fromRGB(160, 160, 175)
PlayerSearch.Text = ""
PlayerSearch.ClearTextOnFocus = false
PlayerSearch.TextColor3 = Color3.fromRGB(255, 255, 255)
PlayerSearch.Font = Enum.Font.SourceSans
PlayerSearch.TextSize = 15
PlayerSearch.Parent = TPJugadoresPage
local SearchCorner = Instance.new("UICorner")
SearchCorner.CornerRadius = UDim.new(0, 6)
SearchCorner.Parent = PlayerSearch

local SearchStatus = Instance.new("TextLabel")
SearchStatus.Size = UDim2.new(1, -10, 0, 22)
SearchStatus.LayoutOrder = -1
SearchStatus.BackgroundTransparency = 1
SearchStatus.TextColor3 = Color3.fromRGB(160, 160, 175)
SearchStatus.Font = Enum.Font.SourceSans
SearchStatus.TextSize = 13
SearchStatus.TextXAlignment = Enum.TextXAlignment.Left
SearchStatus.Parent = TPJugadoresPage

-- Favoritos de esta ejecución, identificados por usuario.
local favorites = {}
local playerListConnections = {}
updatePlayerList = function(excludedPlayer)
    if closed then return end
    for _, connection in ipairs(playerListConnections) do
        disconnect(connection)
    end
    table.clear(playerListConnections)
    local query = string.lower(PlayerSearch.Text):match("^%s*(.-)%s*$")
    query = query:gsub("^@", "")
    local count = 0
	for _, child in ipairs(TPJugadoresPage:GetChildren()) do
		if child:IsA("Frame") then
			child:Destroy()
		end
	end

    local orderedPlayers = Players:GetPlayers()
    table.sort(orderedPlayers, function(a, b)
        local af, bf = favorites[a.UserId] == true, favorites[b.UserId] == true
        if af ~= bf then return af end
        return string.lower(a.Name) < string.lower(b.Name)
    end)
	for _, player in ipairs(orderedPlayers) do
		local matches = query == "" or string.find(string.lower(player.Name), query, 1, true)
            or string.find(string.lower(player.DisplayName), query, 1, true)
        if player ~= LocalPlayer and player ~= excludedPlayer and matches then
            count = count + 1
			local pItem = Instance.new("Frame")
			pItem.Size = UDim2.new(1, -10, 0, 56)
            pItem.LayoutOrder = count
			pItem.BackgroundColor3 = favorites[player.UserId] and Color3.fromRGB(46, 29, 30) or Color3.fromRGB(30, 22, 29)
			pItem.BorderSizePixel = 0
			pItem.Parent = TPJugadoresPage

			local pCorner = Instance.new("UICorner")
			pCorner.CornerRadius = UDim.new(0, 6)
			pCorner.Parent = pItem
            local cardBorder = Instance.new("UIStroke")
            cardBorder.Color = favorites[player.UserId] and Color3.fromRGB(225, 173, 73) or Color3.fromRGB(116, 49, 69)
            cardBorder.Transparency = favorites[player.UserId] and 0.45 or 0.7
            cardBorder.Thickness = 1
            cardBorder.Parent = pItem

			local statusDot = Instance.new("Frame")
			statusDot.Size = UDim2.new(0, 10, 0, 10)
			statusDot.Position = UDim2.new(0, 8, 0.5, -5)
			statusDot.BorderSizePixel = 0
			statusDot.Parent = pItem

			local dotCorner = Instance.new("UICorner")
			dotCorner.CornerRadius = UDim.new(1, 0)
			dotCorner.Parent = statusDot

			if isEnemy(player) then
				statusDot.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
			else
				statusDot.BackgroundColor3 = Color3.fromRGB(50, 220, 50)
			end

			local pName = Instance.new("TextLabel")
			pName.Size = UDim2.new(1, -122, 1, 0)
			pName.Position = UDim2.new(0, 26, 0, 0)
			pName.BackgroundTransparency = 1
			pName.Text = player.DisplayName .. "\n@" .. player.Name
            pName.TextTruncate = Enum.TextTruncate.AtEnd
			pName.TextColor3 = Color3.fromRGB(255, 255, 255)
			pName.Font = Enum.Font.SourceSansBold
			pName.TextSize = 14
			pName.TextXAlignment = Enum.TextXAlignment.Left
			pName.Parent = pItem

            local favoriteBtn = Instance.new("TextButton")
            favoriteBtn.Name = "Favorite"
            favoriteBtn.Size = UDim2.new(0, 34, 0.8, 0)
            favoriteBtn.Position = UDim2.new(1, -86, 0.1, 0)
            favoriteBtn.BackgroundColor3 = favorites[player.UserId] and Color3.fromRGB(74, 51, 31) or Color3.fromRGB(42, 30, 39)
            favoriteBtn.Text = favorites[player.UserId] and "★" or "☆"
            favoriteBtn.TextColor3 = Color3.fromRGB(255, 210, 65)
            favoriteBtn.TextSize = 23
            favoriteBtn.Font = Enum.Font.SourceSansBold
            favoriteBtn.Parent = pItem
            local favoriteCorner = Instance.new("UICorner")
            favoriteCorner.CornerRadius = UDim.new(0, 4)
            favoriteCorner.Parent = favoriteBtn
            table.insert(playerListConnections, connect(favoriteBtn.Activated, function()
                favorites[player.UserId] = not favorites[player.UserId] or nil
                notify(player.DisplayName .. (favorites[player.UserId] and " añadido a favoritos" or " eliminado de favoritos"))
                updatePlayerList()
            end))

			local tpBtn = Instance.new("TextButton")
			tpBtn.Size = UDim2.new(0, 42, 0.8, 0)
			tpBtn.Position = UDim2.new(1, -46, 0.1, 0)
			tpBtn.BackgroundColor3 = Color3.fromRGB(174, 31, 54)
			tpBtn.Text = "TP"
			tpBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
			tpBtn.Font = Enum.Font.SourceSansBold
			tpBtn.TextSize = 13
			tpBtn.Parent = pItem

			local tpCorner = Instance.new("UICorner")
			tpCorner.CornerRadius = UDim.new(0, 4)
			tpCorner.Parent = tpBtn

			table.insert(playerListConnections, connect(tpBtn.MouseButton1Click, function()
				if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
					teleportTo(player.Character.HumanoidRootPart)
                else
                    notify("Ese jugador todavía no tiene personaje", true)
				end
			end))
		end
	end
    SearchStatus.Text = count == 0 and "Sin resultados" or (tostring(count) .. " jugadores")
end

connect(PlayerSearch:GetPropertyChangedSignal("Text"), function()
    updatePlayerList()
    TPJugadoresPage.CanvasPosition = Vector2.new(0, 0)
end)
connect(Players.PlayerAdded, function() updatePlayerList() end)
connect(Players.PlayerRemoving, updatePlayerList)

task.spawn(function()
	task.wait(0.5)
	if closed then return end
	updatePlayerList()
end)

--------------------------------------------------------------------------------
-- AIMBOT (Solara)
--------------------------------------------------------------------------------
local AIM_FOV = 180
local AIM_SMOOTHNESS = 0.16
local AIM_PART = "Head"
local MOUSE_SENSITIVITY = 0.35

local isMobileAimHolding = false

connect(MobileAimBtn.InputBegan, function(input)
	if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
		isMobileAimHolding = true
	end
end)

connect(MobileAimBtn.InputEnded, function(input)
	if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
		isMobileAimHolding = false
	end
end)

local function hasWeaponEquipped()
	local character = LocalPlayer.Character
	if not character then return false end
	return character:FindFirstChildOfClass("Tool") ~= nil
end

local function isMouseLocked()
	return UserInputService.MouseBehavior == Enum.MouseBehavior.LockCenter
end

local function getClosestEnemyInFOV()
	local closestPlayer = nil
	local shortestDistance = AIM_FOV
	local mousePos = UserInputService:GetMouseLocation()
	local camera = Workspace.CurrentCamera

	for _, player in ipairs(Players:GetPlayers()) do
		if isEnemy(player) and player.Character then
			local targetPart = player.Character:FindFirstChild(AIM_PART) or player.Character:FindFirstChild("HumanoidRootPart")
			
			if targetPart then
				local screenPos, onScreen = camera:WorldToViewportPoint(targetPart.Position)
				
				if onScreen and screenPos.Z > 0 then
					local distance = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
					
					if distance < shortestDistance then
						shortestDistance = distance
						closestPlayer = player
					end
				end
			end
		end
	end
	
	return closestPlayer
end

local aimbotConnection = connect(RunService.RenderStepped, function()
	local holdingLeftClick = UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton1)
	local hasWeapon = hasWeaponEquipped()
	
	if not ((holdingLeftClick and hasWeapon) or isMobileAimHolding) then
		return
	end

	local target = getClosestEnemyInFOV()
	if not target or not target.Character then return end

	local targetPart = target.Character:FindFirstChild(AIM_PART) or target.Character:FindFirstChild("HumanoidRootPart")
	if not targetPart then return end

	local camera = Workspace.CurrentCamera

	if isMouseLocked() then
		local currentCFrame = camera.CFrame
		local targetCFrame = CFrame.lookAt(currentCFrame.Position, targetPart.Position)
		camera.CFrame = currentCFrame:Lerp(targetCFrame, AIM_SMOOTHNESS)
	else
		local screenPos, onScreen = camera:WorldToViewportPoint(targetPart.Position)
		
		if onScreen then
			local mousePos = UserInputService:GetMouseLocation()
			local deltaX = (screenPos.X - mousePos.X) * MOUSE_SENSITIVITY
			local deltaY = (screenPos.Y - mousePos.Y) * MOUSE_SENSITIVITY

			pcall(function()
				if mousemoverel then
					mousemoverel(deltaX, deltaY)
				end
			end)
		end
	end
end)

--------------------------------------------------------------------------------
-- NO RECOIL
--------------------------------------------------------------------------------
local noRecoilConnection
local function enableNoRecoil()
	if noRecoilConnection then
		noRecoilConnection:Disconnect()
	end

	noRecoilConnection = connect(RunService.RenderStepped, function()
		local character = LocalPlayer.Character
		if not character then return end

		local humanoid = character:FindFirstChildOfClass("Humanoid")
		if humanoid then
			humanoid.CameraOffset = Vector3.new(0, 0, 0)
		end
	end)
end

enableNoRecoil()

--------------------------------------------------------------------------------
-- CERRAR
--------------------------------------------------------------------------------
-- Acabado visual: conserva los eventos y la lógica de los controles.
do
    local titleGlow = Instance.new("UIStroke")
    titleGlow.Color = Color3.fromRGB(192, 28, 52)
    titleGlow.Thickness = 1
    titleGlow.Transparency = 0.65
    titleGlow.Parent = Title
    local styled = setmetatable({}, {__mode = "k"})
    local function styleButton(button)
        if styled[button] or not button:IsA("TextButton") then return end
        styled[button] = true
        button.AutoButtonColor = false
        button.BorderSizePixel = 0
        if button.Name == "SpeedSlider" then return end
        local corner = button:FindFirstChildOfClass("UICorner")
        if corner then corner.CornerRadius = UDim.new(0, 8) end
        local outline = Instance.new("UIStroke")
        outline.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        outline.Color = Color3.fromRGB(225, 61, 78)
        outline.Transparency = 0.78
        outline.Thickness = 1
        outline.Parent = button
        local animation
        local function emphasis(transparency, thickness)
            if animation then animation:Cancel() end
            animation = TweenService:Create(outline, TweenInfo.new(0.13), {
                Transparency = transparency, Thickness = thickness,
            })
            animation:Play()
        end
        local bindings = {}
        local function bind(signal, callback)
            table.insert(bindings, connect(signal, callback))
        end
        bind(button.Destroying, function()
            if animation then animation:Cancel() end
            for _, connection in ipairs(bindings) do disconnect(connection) end
            table.clear(bindings)
        end)
        bind(button.MouseEnter, function() emphasis(0.32, 1) end)
        bind(button.MouseLeave, function() emphasis(0.78, 1) end)
        bind(button.InputBegan, function(input)
            if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
                emphasis(0.05, 2)
            end
        end)
        bind(button.InputEnded, function(input)
            if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
                emphasis(0.78, 1)
            end
        end)
    end
    for _, object in ipairs(MainFrame:GetDescendants()) do styleButton(object) end
    connect(MainFrame.DescendantAdded, function(object)
        if not object:IsA("TextButton") then return end
        task.defer(function()
            if not closed and object.Parent then styleButton(object) end
        end)
    end)

    local function addSwitch(button, caption, isOn)
        button.TextTransparency = 1
        local label = Instance.new("TextLabel")
        label.Size = UDim2.new(1, -74, 1, 0)
        label.Position = UDim2.fromOffset(12, 0)
        label.BackgroundTransparency = 1
        label.Text = caption
        label.TextColor3 = Color3.fromRGB(244, 231, 235)
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.Font = Enum.Font.GothamMedium
        label.TextSize = 13
        label.Parent = button
        local track = Instance.new("Frame")
        track.Size = UDim2.fromOffset(46, 24)
        track.Position = UDim2.new(1, -58, 0.5, -12)
        track.BorderSizePixel = 0
        track.Parent = button
        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(1, 0)
        corner.Parent = track
        local dot = Instance.new("Frame")
        dot.Size = UDim2.fromOffset(18, 18)
        dot.BackgroundColor3 = Color3.fromRGB(255, 237, 241)
        dot.BorderSizePixel = 0
        dot.Parent = track
        local dotCorner = Instance.new("UICorner")
        dotCorner.CornerRadius = UDim.new(1, 0)
        dotCorner.Parent = dot
        local motion, tint
        local function refresh()
            if motion then motion:Cancel() end
            if tint then tint:Cancel() end
            local active = isOn(button.Text)
            motion = TweenService:Create(dot, TweenInfo.new(0.18, Enum.EasingStyle.Quad), {
                Position = UDim2.fromOffset(active and 25 or 3, 3),
            })
            tint = TweenService:Create(track, TweenInfo.new(0.18), {
                BackgroundColor3 = active and Color3.fromRGB(206, 38, 65) or Color3.fromRGB(62, 49, 57),
            })
            motion:Play()
            tint:Play()
        end
        connect(button:GetPropertyChangedSignal("Text"), refresh)
        refresh()
    end
    addSwitch(VisulyBtn, "Visuly  ·  V", function(text) return string.find(text, ": ON", 1, true) ~= nil end)
    addSwitch(JumpBtn, "Súper salto  ·  J", function(text) return string.find(text, ": ON", 1, true) ~= nil end)
    local speedCard = HabilidadesPage:FindFirstChild("SpeedControl")
    if speedCard then
        for _, child in ipairs(speedCard:GetChildren()) do
            if child:IsA("TextButton") and (child.Text == "Activar" or child.Text == "Desactivar") then
                addSwitch(child, "Velocidad", function(text) return text == "Desactivar" end)
            end
        end
    end
end


local function closePanel()
    if closed then return end
	closed = true
	for connection in pairs(connections) do
		connection:Disconnect()
	end
	table.clear(connections)
	visulyEnabled = false
	removeHighlights()
	disableSuperJump()
    disableSpeed()
	
	if aimbotConnection then
		aimbotConnection:Disconnect()
		aimbotConnection = nil
	end

	if noRecoilConnection then
		noRecoilConnection:Disconnect()
		noRecoilConnection = nil
	end
	
	ScreenGui:Destroy()
end
Shutdown.OnInvoke = closePanel
connect(CloseBtn.MouseButton1Click, closePanel)
connect(ScreenGui.Destroying, closePanel)

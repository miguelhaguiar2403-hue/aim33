--[[
    ═══════════════════════════════════════════════════════
    🎯 AIMBOT HUB v10  •  UI REFEITA
    - Botão minimizado ARRASTÁVEL (mouse e touch)
    - Janela arrastável, limitada à tela
    - Abas: Aim / ESP / Custom / Tema
    - 3 temas de fundo: Carro (preto/branco/cinza),
      Azul e Sasuke (roxo/roxo escuro)
    - Lógica de aimbot e ESP igual à v9
    ═══════════════════════════════════════════════════════
]]

-- ==================== IMAGENS DE FUNDO ====================
-- Suba cada foto no Roblox (Creator Hub > Imagens) e cole o ID aqui.
-- Exemplo: Carro = "rbxassetid://1234567890"
-- Vazio = usa só o tema de cores (já funciona sem imagem).
local IMAGES = {
    Carro  = "",
    Azul   = "",
    Sasuke = "",
}

-- ==================== SERVIÇOS ====================
local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")
local Workspace        = game:GetService("Workspace")

local LP     = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- ==================== CONFIG ====================
local CFG = {
    Aimbot=false, ShowFOV=true, TeamCheck=false, WallCheck=false,
    ESPBox=false, ESPLine=false, ESPName=false, ESPHealth=false,
    FOV=200, Smooth=0.25, MaxDist=1500, AimPart="Head",
    Key=Enum.KeyCode.E,
    Color=Color3.fromRGB(170,90,255),
}

local ESPCfg = {
    ColorMode="Roxo", LineThickness=1.5, TextSize=13,
    BoxStyle="Caixa", CornerSize=6, ShowDistance=false, TeamColor=false,
    LineOrigin="Baixo",
}

local ColorPresets = {
    Roxo=Color3.fromRGB(170,90,255), Vermelho=Color3.fromRGB(255,80,80),
    Verde=Color3.fromRGB(100,255,130), Azul=Color3.fromRGB(90,160,255),
    Branco=Color3.fromRGB(255,255,255), Amarelo=Color3.fromRGB(255,220,90),
}
local RainbowHue = 0

-- ==================== TEMAS ====================
local Themes = {
    Carro = {
        Label="🚗 Carro • preto, branco e cinza",
        Bg=Color3.fromRGB(10,10,10),    Panel=Color3.fromRGB(26,26,26),
        Btn=Color3.fromRGB(42,42,42),   BtnHov=Color3.fromRGB(78,78,78),
        Acc=Color3.fromRGB(200,200,200),AccB=Color3.fromRGB(255,255,255),
        Text=Color3.fromRGB(235,235,235),Green=Color3.fromRGB(255,255,255),
        Red=Color3.fromRGB(110,110,110),
    },
    Azul = {
        Label="🌌 Azul • noite",
        Bg=Color3.fromRGB(5,10,30),     Panel=Color3.fromRGB(10,22,62),
        Btn=Color3.fromRGB(18,40,100),  BtnHov=Color3.fromRGB(32,68,160),
        Acc=Color3.fromRGB(70,140,255), AccB=Color3.fromRGB(150,195,255),
        Text=Color3.fromRGB(220,235,255),Green=Color3.fromRGB(130,255,200),
        Red=Color3.fromRGB(255,100,120),
    },
    Sasuke = {
        Label="🥷 Sasuke • roxo e roxo escuro",
        Bg=Color3.fromRGB(14,8,26),     Panel=Color3.fromRGB(30,18,50),
        Btn=Color3.fromRGB(48,26,76),   BtnHov=Color3.fromRGB(78,44,122),
        Acc=Color3.fromRGB(170,90,255), AccB=Color3.fromRGB(210,140,255),
        Text=Color3.fromRGB(235,220,255),Green=Color3.fromRGB(140,255,180),
        Red=Color3.fromRGB(255,90,120),
    },
}
local ThemeOrder = {"Carro","Azul","Sasuke"}

local UI = { Theme="Sasuke", ShowBG=true, BGAlpha=0.75 }

local T = {}
for k,v in pairs(Themes.Sasuke) do
    if typeof(v)=="Color3" then T[k]=v end
end

-- Registro: tudo que muda de cor quando troca o tema
local Themed = {}
local Refreshers = {}
local function SyncUI()
    for _,f in ipairs(Refreshers) do pcall(f) end
end
local function Reg(obj,prop,key)
    Themed[#Themed+1] = {obj,prop,key}
    obj[prop] = T[key]
end

-- ==================== ESTADO / CONEXÕES ====================
local aiming = false
local Conns = {}
local function C(signal,fn)
    local c = signal:Connect(fn)
    Conns[#Conns+1] = c
    return c
end

-- ==================== GUI BASE ====================
local SG=Instance.new("ScreenGui")
SG.Name="AimbotHubV10"; SG.ResetOnSpawn=false; SG.IgnoreGuiInset=true
SG.ZIndexBehavior=Enum.ZIndexBehavior.Sibling
SG.Parent=LP:WaitForChild("PlayerGui")

-- Botão minimizado (arrastável)
local OpenBtn=Instance.new("TextButton")
OpenBtn.Size=UDim2.fromOffset(55,55); OpenBtn.Position=UDim2.fromOffset(20,200)
OpenBtn.Text="🎯"; OpenBtn.TextSize=26; OpenBtn.BorderSizePixel=0
OpenBtn.AutoButtonColor=false; OpenBtn.ZIndex=150; OpenBtn.Parent=SG
Reg(OpenBtn,"BackgroundColor3","Panel"); Reg(OpenBtn,"TextColor3","AccB")
Instance.new("UICorner",OpenBtn).CornerRadius=UDim.new(1,0)
local OS=Instance.new("UIStroke",OpenBtn); OS.Thickness=2; Reg(OS,"Color","Acc")

-- Janela principal
-- Tamanho real do espaço da GUI (mais confiável que Camera.ViewportSize no modo janela)
local function ScreenSize()
    local s=SG.AbsoluteSize
    if s.X<100 or s.Y<100 then s=Camera.ViewportSize end
    if s.X<100 or s.Y<100 then s=Vector2.new(1280,720) end
    return s
end

-- Garante que o elemento nunca fique fora da tela (recentraliza se der erro)
local function EnsureOnScreen(target)
    local s=ScreenSize()
    local sz=target.AbsoluteSize
    local p=target.AbsolutePosition-SG.AbsolutePosition
    local x,y
    if p.X~=p.X or p.Y~=p.Y then
        x,y=(s.X-sz.X)/2,(s.Y-sz.Y)/2
    else
        x=math.clamp(p.X,0,math.max(0,s.X-sz.X))
        y=math.clamp(p.Y,0,math.max(0,s.Y-sz.Y))
    end
    target.Position=UDim2.fromOffset(x,y)
end

local vp=ScreenSize()
local W=math.max(220, math.min(430, vp.X-20))
local H=math.max(260, math.min(500, vp.Y-40))
local Frame=Instance.new("Frame")
Frame.Size=UDim2.fromOffset(W,H)
Frame.Position=UDim2.fromOffset((vp.X-W)/2,(vp.Y-H)/2)
Frame.BackgroundColor3=Color3.new(1,1,1)
Frame.BorderSizePixel=0; Frame.Visible=false; Frame.ZIndex=100; Frame.Active=true; Frame.Parent=SG
Instance.new("UICorner",Frame).CornerRadius=UDim.new(0,12)
local FS=Instance.new("UIStroke",Frame); FS.Thickness=2; Reg(FS,"Color","Acc")

local Grad=Instance.new("UIGradient",Frame)
Grad.Rotation=90

-- Imagem de fundo
local BG=Instance.new("ImageLabel")
BG.Size=UDim2.new(1,0,1,0); BG.BackgroundTransparency=1; BG.Image=""
BG.ScaleType=Enum.ScaleType.Crop; BG.ImageTransparency=UI.BGAlpha
BG.ZIndex=100; BG.Visible=false; BG.Parent=Frame
Instance.new("UICorner",BG).CornerRadius=UDim.new(0,12)

-- Barra de título
local TB=Instance.new("Frame")
TB.Size=UDim2.new(1,0,0,42); TB.BackgroundTransparency=0.1; TB.BorderSizePixel=0
TB.ZIndex=101; TB.Parent=Frame
Reg(TB,"BackgroundColor3","Panel")
Instance.new("UICorner",TB).CornerRadius=UDim.new(0,12)
local Mask=Instance.new("Frame")
Mask.Size=UDim2.new(1,0,0,12); Mask.Position=UDim2.new(0,0,1,-12)
Mask.BackgroundTransparency=0.1; Mask.BorderSizePixel=0; Mask.ZIndex=102; Mask.Parent=TB
Reg(Mask,"BackgroundColor3","Panel")

local TL=Instance.new("TextLabel")
TL.Size=UDim2.new(1,-130,1,0); TL.Position=UDim2.new(0,15,0,0); TL.BackgroundTransparency=1
TL.Text="🎯 AIMBOT HUB v10"; TL.TextSize=17; TL.Font=Enum.Font.GothamBold
TL.TextXAlignment=Enum.TextXAlignment.Left; TL.ZIndex=103; TL.Parent=TB
Reg(TL,"TextColor3","AccB")

local function TitleBtn(txt,xOff,key)
    local b=Instance.new("TextButton")
    b.Size=UDim2.fromOffset(28,28); b.Position=UDim2.new(1,xOff,0,7)
    b.Text=txt; b.TextSize=15; b.Font=Enum.Font.GothamBold
    b.BorderSizePixel=0; b.AutoButtonColor=true; b.ZIndex=103; b.Parent=TB
    Reg(b,"BackgroundColor3",key); Reg(b,"TextColor3","Text")
    Instance.new("UICorner",b).CornerRadius=UDim.new(0,6)
    return b
end
local CB  = TitleBtn("✕",-36,"Red")    -- descarregar
local MinB= TitleBtn("—",-70,"Btn")    -- minimizar
local ThB = TitleBtn("🎨",-104,"Btn")  -- trocar tema

-- ==================== ARRASTAR (mouse + touch) ====================
local function MakeDraggable(handle,target,onTap)
    local dragging,moved,startIn,startPos=false,false,nil,nil
    handle.InputBegan:Connect(function(i)
        if i.UserInputType==Enum.UserInputType.MouseButton1
        or i.UserInputType==Enum.UserInputType.Touch then
            dragging=true; moved=false
            startIn=i.Position; startPos=target.AbsolutePosition-SG.AbsolutePosition
            local conn
            conn=i.Changed:Connect(function()
                if i.UserInputState==Enum.UserInputState.End then
                    conn:Disconnect()
                    if not dragging then return end
                    dragging=false
                    if moved then EnsureOnScreen(target) end
                    if not moved and onTap then onTap() end
                end
            end)
        end
    end)
    C(UserInputService.InputChanged,function(i)
        if not dragging then return end
        if i.UserInputType==Enum.UserInputType.MouseMovement
        or i.UserInputType==Enum.UserInputType.Touch then
            local d=i.Position-startIn
            if d.Magnitude>6 then moved=true end
            if moved then
                local v=ScreenSize()
                local sz=target.AbsoluteSize
                local x=math.clamp(startPos.X+d.X,0,math.max(0,v.X-sz.X))
                local y=math.clamp(startPos.Y+d.Y,0,math.max(0,v.Y-sz.Y))
                target.Position=UDim2.fromOffset(x,y)
            end
        end
    end)
end

-- ==================== TEMA: aplicar ====================
local function UpdateBG()
    local id=IMAGES[UI.Theme]
    BG.Image=id or ""
    BG.ImageTransparency=UI.BGAlpha
    BG.Visible=(UI.ShowBG and id~=nil and id~="")
end

local function ApplyTheme(name)
    local th=Themes[name]; if not th then return end
    UI.Theme=name
    for k,v in pairs(th) do
        if typeof(v)=="Color3" then T[k]=v end
    end
    for _,r in ipairs(Themed) do r[1][r[2]]=T[r[3]] end
    Grad.Color=ColorSequence.new(T.Bg,T.Panel)
    UpdateBG()
    for _,f in ipairs(Refreshers) do f() end
end

-- ==================== ABAS / PÁGINAS ====================
local Pages,TabBtns={}, {}
local ActivePage="Aim"
local CurPage

local function NewPage(name)
    local p=Instance.new("ScrollingFrame")
    p.Size=UDim2.new(1,-20,1,-96); p.Position=UDim2.new(0,10,0,86)
    p.BackgroundTransparency=1; p.BorderSizePixel=0; p.ScrollBarThickness=4
    p.CanvasSize=UDim2.new(); p.AutomaticCanvasSize=Enum.AutomaticSize.Y
    p.ZIndex=101; p.Visible=false; p.Parent=Frame
    Reg(p,"ScrollBarImageColor3","Acc")
    local l=Instance.new("UIListLayout",p)
    l.Padding=UDim.new(0,6); l.SortOrder=Enum.SortOrder.LayoutOrder
    Pages[name]=p
    return p
end

local function RefreshTabs()
    for name,b in pairs(TabBtns) do
        local on=(name==ActivePage)
        b.BackgroundColor3=on and T.Acc or T.Btn
        b.TextColor3=on and T.Bg or T.Text
    end
end
Refreshers[#Refreshers+1]=RefreshTabs

local function ShowPage(name)
    ActivePage=name
    for n,p in pairs(Pages) do p.Visible=(n==name) end
    RefreshTabs()
end

local TabBar=Instance.new("Frame")
TabBar.Size=UDim2.new(1,-20,0,30); TabBar.Position=UDim2.new(0,10,0,48)
TabBar.BackgroundTransparency=1; TabBar.ZIndex=101; TabBar.Parent=Frame
local TL2=Instance.new("UIListLayout",TabBar)
TL2.FillDirection=Enum.FillDirection.Horizontal; TL2.Padding=UDim.new(0,4)
TL2.SortOrder=Enum.SortOrder.LayoutOrder

for idx,t in ipairs({{"Aim","🎯 Aim"},{"ESP","👁️ ESP"},{"Custom","🎨 Custom"},{"Tema","⚙️ Tema"}}) do
    NewPage(t[1])
    local b=Instance.new("TextButton")
    b.Size=UDim2.new(0.25,-3,1,0); b.Text=t[2]; b.TextSize=12
    b.Font=Enum.Font.GothamBold; b.BorderSizePixel=0; b.AutoButtonColor=false
    b.LayoutOrder=idx; b.ZIndex=102; b.Parent=TabBar
    Instance.new("UICorner",b).CornerRadius=UDim.new(0,7)
    b.MouseButton1Click:Connect(function() ShowPage(t[1]) end)
    TabBtns[t[1]]=b
end

-- ==================== HELPERS DE UI ====================
local function MakeBtn(txt,key,fn)
    key=key or "Btn"
    local b=Instance.new("TextButton")
    b.Size=UDim2.new(1,-10,0,34); b.BackgroundTransparency=0.15
    b.Text="  "..txt; b.TextSize=14; b.Font=Enum.Font.GothamMedium
    b.TextXAlignment=Enum.TextXAlignment.Left; b.BorderSizePixel=0
    b.AutoButtonColor=false; b.ZIndex=102; b.Parent=CurPage
    Reg(b,"BackgroundColor3",key); Reg(b,"TextColor3","Text")
    Instance.new("UICorner",b).CornerRadius=UDim.new(0,7)
    local s=Instance.new("UIStroke",b); s.Thickness=1; s.Transparency=0.6
    Reg(s,"Color","Acc")
    local function hov()
        return key=="Btn" and T.BtnHov or T[key]:Lerp(Color3.new(1,1,1),0.15)
    end
    b.MouseEnter:Connect(function()
        TweenService:Create(b,TweenInfo.new(0.15),{BackgroundColor3=hov()}):Play()
    end)
    b.MouseLeave:Connect(function()
        TweenService:Create(b,TweenInfo.new(0.15),{BackgroundColor3=T[key]}):Play()
    end)
    b.MouseButton1Click:Connect(function() fn(b) end)
    return b
end

local function MakeToggle(txt,tbl,key,onChange)
    local b
    local function refresh()
        local on=tbl[key]
        b.Text=(on and "  🟢 " or "  🔴 ")..txt
        b.TextColor3=on and T.Green or T.Text
    end
    b=MakeBtn(txt,"Btn",function()
        tbl[key]=not tbl[key]
        refresh()
        if onChange then onChange() end
    end)
    Refreshers[#Refreshers+1]=refresh
    refresh()
    return b
end

local function Section(txt)
    local l=Instance.new("TextLabel")
    l.Size=UDim2.new(1,-10,0,26); l.BackgroundTransparency=1
    l.Text="  ── "..txt.." ──"; l.TextSize=13
    l.Font=Enum.Font.GothamBold; l.TextXAlignment=Enum.TextXAlignment.Left
    l.ZIndex=102; l.Parent=CurPage
    Reg(l,"TextColor3","Acc")
end

-- ==================== FOV CIRCLE ====================
local FovC=Instance.new("Frame")
FovC.AnchorPoint=Vector2.new(0.5,0.5)
FovC.Position=UDim2.new(0.5,0,0.5,0)
FovC.Size=UDim2.new(0,CFG.FOV*2,0,CFG.FOV*2)
FovC.BackgroundTransparency=1; FovC.BorderSizePixel=0
FovC.ZIndex=5; FovC.Visible=false; FovC.Parent=SG
Instance.new("UICorner",FovC).CornerRadius=UDim.new(1,0)
local FovS=Instance.new("UIStroke",FovC)
FovS.Thickness=2; FovS.Transparency=0.3

-- ==================== UTIL ====================
local function ValidTarget(plr)
    if plr==LP then return false end
    if CFG.TeamCheck and plr.Team==LP.Team then return false end
    local c=plr.Character
    if not c then return false end
    local h=c:FindFirstChildOfClass("Humanoid")
    if not h or h.Health<=0 then return false end
    return c:FindFirstChild(CFG.AimPart) ~= nil
end

local function InFov(pos, radius)
    local sp,on=Camera:WorldToViewportPoint(pos)
    if not on then return false,math.huge end
    local ctr=Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    local d=(Vector2.new(sp.X,sp.Y)-ctr).Magnitude
    return d<=radius, d
end

local function IsVisible(targetPart)
    if not CFG.WallCheck then return true end
    local o=Camera.CFrame.Position
    local params=RaycastParams.new()
    params.FilterType=Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances={LP.Character, Camera}
    local r=Workspace:Raycast(o, targetPart.Position-o, params)
    if r then return r.Instance:IsDescendantOf(targetPart.Parent) end
    return true
end

-- ==================== PÁGINA: AIM ====================
CurPage=Pages.Aim
Section("🎯 AIMBOT")
MakeToggle("Aimbot (E)", CFG, "Aimbot")
MakeToggle("Mostrar FOV", CFG, "ShowFOV")
MakeToggle("Team Check", CFG, "TeamCheck")
MakeToggle("Wall Check", CFG, "WallCheck")
MakeBtn("🎯 FOV: "..CFG.FOV,nil,function(b)
    CFG.FOV=CFG.FOV+50; if CFG.FOV>600 then CFG.FOV=100 end
    b.Text="  🎯 FOV: "..CFG.FOV
end)
MakeBtn("💧 Smooth: "..string.format("%.2f",CFG.Smooth),nil,function(b)
    CFG.Smooth=CFG.Smooth+0.05
    if CFG.Smooth>1.0 then CFG.Smooth=0.05 end
    b.Text="  💧 Smooth: "..string.format("%.2f",CFG.Smooth)
end)
MakeBtn("📏 Dist: "..CFG.MaxDist,nil,function(b)
    CFG.MaxDist=CFG.MaxDist+500; if CFG.MaxDist>5000 then CFG.MaxDist=500 end
    b.Text="  📏 Dist: "..CFG.MaxDist
end)
MakeBtn("🎯 Mira: "..CFG.AimPart,nil,function(b)
    local list={"Head","HumanoidRootPart","UpperTorso"}
    local i=table.find(list,CFG.AimPart) or 1
    i=i%#list+1
    CFG.AimPart=list[i]
    b.Text="  🎯 Mira: "..CFG.AimPart
end)

-- ==================== PÁGINA: ESP ====================
CurPage=Pages.ESP
Section("👁️ ESP")
MakeToggle("ESP Box", CFG, "ESPBox")
MakeToggle("ESP Line", CFG, "ESPLine")
MakeToggle("ESP Nome", CFG, "ESPName")
MakeToggle("ESP Vida", CFG, "ESPHealth")

-- ==================== PÁGINA: CUSTOM ====================
CurPage=Pages.Custom
Section("🎨 ESP CUSTOM")
MakeBtn("🎨 Cor ESP: "..ESPCfg.ColorMode,nil,function(b)
    local list={"Roxo","Vermelho","Verde","Azul","Branco","Amarelo","Rainbow"}
    local i=table.find(list,ESPCfg.ColorMode) or 1
    i=i%#list+1
    ESPCfg.ColorMode=list[i]
    b.Text="  🎨 Cor ESP: "..ESPCfg.ColorMode
end)
MakeBtn("📏 Espessura: "..string.format("%.1f",ESPCfg.LineThickness),nil,function(b)
    ESPCfg.LineThickness=ESPCfg.LineThickness+0.5
    if ESPCfg.LineThickness>5 then ESPCfg.LineThickness=0.5 end
    b.Text="  📏 Espessura: "..string.format("%.1f",ESPCfg.LineThickness)
end)
MakeBtn("🔤 Texto: "..ESPCfg.TextSize,nil,function(b)
    ESPCfg.TextSize=ESPCfg.TextSize+2
    if ESPCfg.TextSize>24 then ESPCfg.TextSize=9 end
    b.Text="  🔤 Texto: "..ESPCfg.TextSize
end)
MakeBtn("⬜ Estilo: "..ESPCfg.BoxStyle,nil,function(b)
    ESPCfg.BoxStyle=(ESPCfg.BoxStyle=="Caixa") and "Cantos" or "Caixa"
    b.Text="  ⬜ Estilo: "..ESPCfg.BoxStyle
end)
MakeBtn("📍 Linha sai de: "..ESPCfg.LineOrigin,nil,function(b)
    local list={"Baixo","Meio","Topo"}
    local i=table.find(list,ESPCfg.LineOrigin) or 1
    ESPCfg.LineOrigin=list[i%#list+1]
    b.Text="  📍 Linha sai de: "..ESPCfg.LineOrigin
end)
MakeBtn("📐 Canto: "..ESPCfg.CornerSize.."px",nil,function(b)
    ESPCfg.CornerSize=ESPCfg.CornerSize+2
    if ESPCfg.CornerSize>20 then ESPCfg.CornerSize=4 end
    b.Text="  📐 Canto: "..ESPCfg.CornerSize.."px"
end)
MakeToggle("Mostrar Distância", ESPCfg, "ShowDistance")
MakeToggle("Cor do Time", ESPCfg, "TeamColor")

-- ==================== PÁGINA: TEMA ====================
CurPage=Pages.Tema
Section("🖼️ FUNDO DO HUB")
local Row=Instance.new("Frame")
Row.Size=UDim2.new(1,-10,0,46); Row.BackgroundTransparency=1
Row.ZIndex=102; Row.Parent=CurPage
local RL=Instance.new("UIListLayout",Row)
RL.FillDirection=Enum.FillDirection.Horizontal; RL.Padding=UDim.new(0,6)
RL.SortOrder=Enum.SortOrder.LayoutOrder
local ThemeIcons={Carro="🚗",Azul="🌌",Sasuke="🥷"}
for idx,n in ipairs(ThemeOrder) do
    local b=Instance.new("TextButton")
    b.Size=UDim2.new(1/3,-4,1,0); b.Text=ThemeIcons[n]; b.TextSize=24
    b.BorderSizePixel=0; b.AutoButtonColor=false; b.BackgroundTransparency=0.15
    b.LayoutOrder=idx; b.ZIndex=103; b.Parent=Row
    Instance.new("UICorner",b).CornerRadius=UDim.new(0,9)
    local s=Instance.new("UIStroke",b); s.Thickness=2
    b.MouseButton1Click:Connect(function() ApplyTheme(n) end)
    Refreshers[#Refreshers+1]=function()
        local on=(UI.Theme==n)
        b.BackgroundColor3=on and T.Acc or T.Btn
        s.Color=T.Acc; s.Transparency=on and 0 or 0.7
    end
end
MakeToggle("Mostrar imagem de fundo", UI, "ShowBG", UpdateBG)
MakeBtn("🖼️ Visibilidade: "..math.floor((1-UI.BGAlpha)*100+0.5).."%",nil,function(b)
    UI.BGAlpha=UI.BGAlpha-0.1
    if UI.BGAlpha<0.45 then UI.BGAlpha=0.9 end
    UpdateBG()
    b.Text="  🖼️ Visibilidade: "..math.floor((1-UI.BGAlpha)*100+0.5).."%"
end)

local Note=Instance.new("TextLabel")
Note.Size=UDim2.new(1,-10,0,0); Note.AutomaticSize=Enum.AutomaticSize.Y
Note.BackgroundTransparency=1; Note.TextWrapped=true; Note.TextSize=12
Note.Font=Enum.Font.Gotham; Note.TextXAlignment=Enum.TextXAlignment.Left
Note.ZIndex=102; Note.Parent=CurPage
Reg(Note,"TextColor3","Text")
Refreshers[#Refreshers+1]=function()
    local id=IMAGES[UI.Theme]
    Note.Text=(id and id~="") and "  Imagem carregada ✔"
        or "  ⚠ Sem imagem neste tema: cole o ID em IMAGES (topo do script). As cores já funcionam."
end

Section("⚙️ SISTEMA")
MakeBtn("🛑 Destravar Câmera","Red",function(b)
    CFG.Aimbot=false
    aiming=false
    SyncUI()
    b.Text="  ✅ Destravado"
    task.delay(1,function() b.Text="  🛑 Destravar Câmera" end)
end)

-- ==================== ESP RENDERER ====================
local ESP={}
local function BuildESP(plr)
    if plr==LP or ESP[plr] then return end
    local d={}
    d.box=Instance.new("Frame",SG)
    d.box.BackgroundTransparency=1; d.box.BorderSizePixel=0
    d.box.Visible=false; d.box.ZIndex=3
    Instance.new("UICorner",d.box).CornerRadius=UDim.new(0,3)
    d.stroke=Instance.new("UIStroke",d.box)
    d.stroke.Color=CFG.Color; d.stroke.Thickness=1.5

    d.corners={}
    for i=1,4 do
        local c=Instance.new("Frame",SG)
        c.BackgroundColor3=CFG.Color; c.BorderSizePixel=0
        c.Visible=false; c.ZIndex=3
        d.corners[i]=c
    end

    d.line=Instance.new("Frame",SG)
    d.line.BackgroundColor3=CFG.Color; d.line.BorderSizePixel=0
    d.line.Visible=false; d.line.AnchorPoint=Vector2.new(0.5,0); d.line.ZIndex=3

    d.name=Instance.new("TextLabel",SG)
    d.name.BackgroundTransparency=1; d.name.TextColor3=CFG.Color
    d.name.TextSize=13; d.name.Font=Enum.Font.GothamBold
    d.name.TextStrokeTransparency=0.4; d.name.Visible=false; d.name.ZIndex=4
    d.name.Text=plr.Name

    d.hpBg=Instance.new("Frame",SG)
    d.hpBg.BackgroundColor3=Color3.fromRGB(30,30,30)
    d.hpBg.BorderSizePixel=0; d.hpBg.Visible=false; d.hpBg.ZIndex=3

    d.hpFill=Instance.new("Frame",d.hpBg)
    d.hpFill.BackgroundColor3=Color3.fromRGB(120,255,120)
    d.hpFill.BorderSizePixel=0; d.hpFill.Size=UDim2.new(1,0,1,0); d.hpFill.ZIndex=4

    ESP[plr]=d
end

local function KillESP(plr)
    if not ESP[plr] then return end
    for _,o in pairs(ESP[plr]) do
        if typeof(o)=="Instance" and o.Destroy then o:Destroy()
        elseif typeof(o)=="table" then
            for _,c in pairs(o) do
                if c.Destroy then c:Destroy() end
            end
        end
    end
    ESP[plr]=nil
end

C(Players.PlayerAdded,BuildESP)
C(Players.PlayerRemoving,KillESP)
for _,p in pairs(Players:GetPlayers()) do BuildESP(p) end

local function GetESPColor(plr)
    if ESPCfg.TeamColor and plr and plr.Team then
        return plr.Team.TeamColor.Color
    end
    if ESPCfg.ColorMode=="Rainbow" then
        return Color3.fromHSV(RainbowHue, 0.8, 1)
    end
    return ColorPresets[ESPCfg.ColorMode] or Color3.fromRGB(170,90,255)
end

-- Cantos de uma caixa 3D (usados para montar a caixa 2D que cobre o corpo todo)
local CORNER_SIGNS={}
for _,x in ipairs({-1,1}) do
    for _,y in ipairs({-1,1}) do
        for _,z in ipairs({-1,1}) do
            CORNER_SIGNS[#CORNER_SIGNS+1]=Vector3.new(x,y,z)
        end
    end
end

-- Caixa 2D baseada no corpo inteiro (não depende de HumanoidRootPart/Head
-- e continua aparecendo com o player parcialmente fora da tela)
local function GetBox2D(c)
    local ok,cf,ext=pcall(function() return c:GetBoundingBox() end)
    if not ok or not cf then return nil end
    local center=cf.Position
    local h=math.clamp(ext.Y,3.5,9)
    local w=h*0.55
    local half=Vector3.new(w/2,h/2,w/2)
    local minX,minY,maxX,maxY=math.huge,math.huge,-math.huge,-math.huge
    local any=false
    for _,s in ipairs(CORNER_SIGNS) do
        local sp=Camera:WorldToViewportPoint(center+half*s)
        if sp.Z>0 then
            any=true
            if sp.X<minX then minX=sp.X end
            if sp.Y<minY then minY=sp.Y end
            if sp.X>maxX then maxX=sp.X end
            if sp.Y>maxY then maxY=sp.Y end
        end
    end
    if not any then return nil end
    return minX,minY,math.max(maxX-minX,4),math.max(maxY-minY,6),center
end

local function HideESP(d)
    d.box.Visible=false; d.line.Visible=false
    d.name.Visible=false; d.hpBg.Visible=false
    for i=1,4 do d.corners[i].Visible=false end
end

C(RunService.RenderStepped,function()
    Camera=Workspace.CurrentCamera or Camera
    if ESPCfg.ColorMode=="Rainbow" then
        RainbowHue=(RainbowHue+0.005)%1
    end

    local show=CFG.ESPBox or CFG.ESPLine or CFG.ESPName or CFG.ESPHealth

    for plr,d in pairs(ESP) do
        local c=plr.Character
        local hum=c and c:FindFirstChildOfClass("Humanoid")
        local alive=c and (not hum or hum.Health>0)
        local bx,by,bw,bh,center
        if show and alive then
            bx,by,bw,bh,center=GetBox2D(c)
        end

        if bx then
            local color=GetESPColor(plr)

            if ESPCfg.BoxStyle=="Caixa" then
                d.box.Visible=CFG.ESPBox
                d.box.Size=UDim2.new(0,bw,0,bh)
                d.box.Position=UDim2.new(0,bx,0,by)
                d.stroke.Color=color
                d.stroke.Thickness=ESPCfg.LineThickness
                for i=1,4 do d.corners[i].Visible=false end
            else
                d.box.Visible=false
                local cs=math.min(ESPCfg.CornerSize,bw/2,bh/2)
                local cth=math.max(1,math.floor(ESPCfg.LineThickness))
                local pos={
                    {bx,by},{bx+bw-cs,by},
                    {bx,by+bh-cth},{bx+bw-cs,by+bh-cth},
                }
                for i=1,4 do
                    local cc=d.corners[i]
                    cc.Visible=CFG.ESPBox
                    cc.Position=UDim2.new(0,pos[i][1],0,pos[i][2])
                    cc.Size=UDim2.new(0,cs,0,cth)
                    cc.BackgroundColor3=color
                end
            end

            d.name.Visible=CFG.ESPName
            d.name.Size=UDim2.new(0,200,0,16)
            d.name.Position=UDim2.new(0,bx+bw/2-100,0,by-20)
            d.name.TextColor3=color
            d.name.TextSize=ESPCfg.TextSize
            if ESPCfg.ShowDistance then
                local dist=math.floor((center-Camera.CFrame.Position).Magnitude)
                d.name.Text=plr.Name.." ["..dist.."m]"
            else
                d.name.Text=plr.Name
            end

            local frac=hum and math.clamp(hum.Health/math.max(hum.MaxHealth,1),0,1) or 1
            d.hpBg.Visible=CFG.ESPHealth
            d.hpBg.Size=UDim2.new(0,4,0,bh)
            d.hpBg.Position=UDim2.new(0,bx-8,0,by)
            d.hpFill.Size=UDim2.new(1,0,frac,0)
            d.hpFill.Position=UDim2.new(0,0,1-frac,0)

            d.line.Visible=CFG.ESPLine
            local vs=Camera.ViewportSize
            local from,to
            if ESPCfg.LineOrigin=="Topo" then
                from=Vector2.new(vs.X/2,0)
                to=Vector2.new(bx+bw/2,by)          -- cabeça
            elseif ESPCfg.LineOrigin=="Meio" then
                from=Vector2.new(vs.X/2,vs.Y/2)
                to=Vector2.new(bx+bw/2,by+bh/2)     -- centro do corpo
            else
                from=Vector2.new(vs.X/2,vs.Y)
                to=Vector2.new(bx+bw/2,by+bh)       -- pés
            end
            local dir=to-from
            local len=dir.Magnitude
            if len<1 then
                d.line.Visible=false
            else
                -- linha desenhada pelo ponto médio (gira certo em qualquer pivô)
                local mid=(from+to)/2
                d.line.AnchorPoint=Vector2.new(0.5,0.5)
                d.line.Size=UDim2.new(0,len,0,ESPCfg.LineThickness)
                d.line.Position=UDim2.new(0,mid.X,0,mid.Y)
                d.line.Rotation=math.deg(math.atan2(dir.Y,dir.X))
                d.line.BackgroundColor3=color
            end
        else
            HideESP(d)
        end
    end
end)

-- ==================== INPUTS ====================
C(UserInputService.InputBegan,function(input,gpe)
    -- só ignora se estiver digitando num campo de texto (chat etc.);
    -- jogos que usam a tecla E marcam "gpe" e antes travavam o aimbot
    if UserInputService:GetFocusedTextBox() then return end
    if input.KeyCode==CFG.Key then aiming=true end
end)
C(UserInputService.InputEnded,function(input)
    if input.KeyCode==CFG.Key then aiming=false end
end)
C(LP.CharacterAdded,function()
    aiming=false
end)

-- ==================== AIMBOT ====================
C(RunService.RenderStepped,function()
    FovC.Visible=CFG.ShowFOV
    FovC.Size=UDim2.new(0,CFG.FOV*2,0,CFG.FOV*2)
    FovS.Color=CFG.Aimbot and T.Acc or T.Red
end)

-- Alvo travado: o aimbot segue o mesmo player até ele ficar inválido
local lockedPlr=nil

local function PickTarget()
    local bestPart,bestPlr,bestD=nil,nil,math.huge
    for _,plr in pairs(Players:GetPlayers()) do
        if ValidTarget(plr) then
            local p=plr.Character[CFG.AimPart]
            local d3=(p.Position-Camera.CFrame.Position).Magnitude
            if d3<=CFG.MaxDist then
                local ok,fd=InFov(p.Position, CFG.FOV)
                if ok and fd<bestD and IsVisible(p) then
                    bestD=fd; bestPart=p; bestPlr=plr
                end
            end
        end
    end
    return bestPlr,bestPart
end

local lastLook=nil
local SNAP=math.rad(4) -- perto do alvo, a mira gruda 100%

-- Prioridade "Last": roda depois de câmera, personagens e animações,
-- então a mira usa a posição final do alvo no frame (sem atraso).
RunService:BindToRenderStep("PurpleAimbot", Enum.RenderPriority.Last.Value, function()
    Camera=Workspace.CurrentCamera or Camera
    if not CFG.Aimbot or not aiming then
        lockedPlr=nil; lastLook=nil
        return
    end

    -- continua no alvo atual enquanto ele for válido
    local part
    if lockedPlr and ValidTarget(lockedPlr) then
        local p=lockedPlr.Character[CFG.AimPart]
        if (p.Position-Camera.CFrame.Position).Magnitude<=CFG.MaxDist and IsVisible(p) then
            part=p
        end
    end

    -- senão escolhe o mais perto do centro da tela
    if not part then
        lockedPlr,part=PickTarget()
    end
    if not part then lastLook=nil; return end

    local camPos=Camera.CFrame.Position
    local to=part.Position-camPos
    if to.Magnitude<0.01 then return end
    local want=to.Unit

    -- A mira parte da última direção usada (não da câmera do jogo) e
    -- converge pro alvo: o Smooth controla a velocidade de chegada e, ao
    -- chegar perto, gruda 100% e acompanha o alvo sem ficar pra trás.
    local cur=lastLook or Camera.CFrame.LookVector
    local ang=math.acos(math.clamp(cur:Dot(want),-1,1))
    local alpha=(ang<=SNAP) and 1 or math.clamp(1-CFG.Smooth,0.05,1)
    local nl=cur:Lerp(want,alpha)
    if nl.Magnitude<0.001 then nl=want else nl=nl.Unit end
    lastLook=nl
    Camera.CFrame=CFrame.new(camPos,camPos+nl)
end)

-- ==================== DESCARREGAR ====================
local function Unload()
    CFG.Aimbot=false; aiming=false
    pcall(function() RunService:UnbindFromRenderStep("PurpleAimbot") end)
    for _,c in ipairs(Conns) do pcall(function() c:Disconnect() end) end
    SG:Destroy()
end

C(LP.AncestryChanged,function()
    if not LP:IsDescendantOf(game) then
        pcall(function() RunService:UnbindFromRenderStep("PurpleAimbot") end)
    end
end)

-- ==================== ABRIR / MINIMIZAR / TEMA ====================
-- Janela: arrasta pela barra de título
MakeDraggable(TB, Frame)
-- Botão minimizado: arrasta; toque rápido abre/fecha
-- Hub aberto = botão 🎯 escondido (evita clicar nele sem querer por cima da janela)
local function SetHub(open)
    Frame.Visible=open
    OpenBtn.Visible=not open
    if open then EnsureOnScreen(Frame) end
end

MakeDraggable(OpenBtn, OpenBtn, function() SetHub(true) end)
MinB.MouseButton1Click:Connect(function() SetHub(false) end)

-- ✕ descarrega tudo: pede um segundo toque para confirmar
local confirmUnload=false
CB.MouseButton1Click:Connect(function()
    if confirmUnload then Unload(); return end
    confirmUnload=true
    CB.Text="?"
    task.delay(2,function()
        confirmUnload=false
        if CB.Parent then CB.Text="✕" end
    end)
end)
ThB.MouseButton1Click:Connect(function()
    local i=table.find(ThemeOrder,UI.Theme) or 1
    ApplyTheme(ThemeOrder[i%#ThemeOrder+1])
end)

-- ==================== INICIAR ====================
ShowPage("Aim")
ApplyTheme(UI.Theme)

task.spawn(function()
    local N=Instance.new("TextLabel",SG)
    N.Size=UDim2.new(0,440,0,40); N.Position=UDim2.new(0.5,-220,0,20)
    N.BackgroundTransparency=0; N.Text="🎯 v10 | [E] Aimbot • 🎯 arrastável"
    N.TextSize=15; N.Font=Enum.Font.GothamBold; N.BorderSizePixel=0; N.ZIndex=200
    N.BackgroundColor3=T.Panel; N.TextColor3=T.AccB
    Instance.new("UICorner",N).CornerRadius=UDim.new(0,8)
    task.wait(3.5)
    TweenService:Create(N,TweenInfo.new(0.5),{BackgroundTransparency=1,TextTransparency=1}):Play()
    task.wait(0.6); N:Destroy()
end)

print("🎯 AIMBOT HUB v10 carregado. [E] Aimbot")

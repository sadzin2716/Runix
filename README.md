-- 1. Biblioteca MaterialLua 
local Material = loadstring(game:HttpGet("https://raw.githubusercontent.com/Kinlei/MaterialLua/master/Module.lua"))()

-- 2. Serviços e variáveis
local Players, RunService, LocalPlayer = game:GetService("Players"), game:GetService("RunService"), game:GetService("Players").LocalPlayer
local Mouse = LocalPlayer:GetMouse()
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local camera = workspace.CurrentCamera

local config = { espEnabled = true, enemyColor = Color3.fromRGB(255,0,0), teamColor = Color3.fromRGB(0,255,0) }
local hitboxConfig = { enabled = false, size = Vector3.new(7,7,7), normalSize = Vector3.new(2,2,1) }

-- 3. Funções ESP / Hitbox
local function updateAll()
    for _, pl in pairs(Players:GetPlayers()) do
        if pl ~= LocalPlayer and pl.Character then
            local hl = pl.Character:FindFirstChild("ESP_Highlight")
            if hl then
                local col = (pl.Team == LocalPlayer.Team) and config.teamColor or config.enemyColor
                hl.FillColor, hl.OutlineColor, hl.Enabled = col, col, config.espEnabled
            end
            local hrp = pl.Character:FindFirstChild("HumanoidRootPart")
            if hrp then
                if hitboxConfig.enabled and pl.Team ~= LocalPlayer.Team then
                    hrp.Size, hrp.Transparency, hrp.Material = hitboxConfig.size, 0.6, Enum.Material.ForceField
                else
                    hrp.Size, hrp.Transparency, hrp.Material = hitboxConfig.normalSize, 1, Enum.Material.Plastic
                end
                hrp.CanCollide = false
            end
        end
    end
end

local function tryCreateESP(pl, char)
    if char:FindFirstChild("ESP_Highlight") then return end
    local hl = Instance.new("Highlight", char)
    hl.Name, hl.FillTransparency, hl.OutlineTransparency = "ESP_Highlight", 0.5, 0
    hl.FillColor = (pl.Team == LocalPlayer.Team) and config.teamColor or config.enemyColor
    hl.OutlineColor = hl.FillColor
    hl.Enabled = config.espEnabled
end

local function applyToPlayer(pl)
    if pl == LocalPlayer then return end
    pl.CharacterAdded:Connect(function(char)
        repeat task.wait() until char:FindFirstChild("HumanoidRootPart")
        tryCreateESP(pl, char)
        updateAll()
    end)
    if pl.Character then
        tryCreateESP(pl, pl.Character)
        updateAll()
    end
end
for _, pl in pairs(Players:GetPlayers()) do applyToPlayer(pl) end
Players.PlayerAdded:Connect(applyToPlayer)
task.spawn(function() while true do if hitboxConfig.enabled then updateAll() end RunService.Heartbeat:Wait() end end)

-- 4. Aimbot com FOV e Lerp
local aiming = false
local aimbotFOV = 100

local function isAlive(plr)
    return plr.Character and plr.Character:FindFirstChild("Head") and plr.Character:FindFirstChild("Humanoid") and plr.Character.Humanoid.Health > 0
end

local function getClosestTarget()
    local closest, shortestDist = nil, math.huge
    local mousePos = Vector2.new(Mouse.X, Mouse.Y)
    for _, pl in pairs(Players:GetPlayers()) do
        if pl ~= LocalPlayer and isAlive(pl) then
            local headPos, onScreen = camera:WorldToViewportPoint(pl.Character.Head.Position)
            if onScreen then
                local screenPos = Vector2.new(headPos.X, headPos.Y)
                local dist = (mousePos - screenPos).Magnitude
                if dist < shortestDist and dist <= aimbotFOV then
                    shortestDist = dist
                    closest = pl
                end
            end
        end
    end
    return closest
end

local Drawing = Drawing
local fovCircle = Drawing.new("Circle")
fovCircle.Visible = false
fovCircle.Radius = aimbotFOV
fovCircle.Thickness = 2
fovCircle.Transparency = 0.5
fovCircle.Color = Color3.fromRGB(255, 255, 0)
fovCircle.Filled = false

RunService.RenderStepped:Connect(function()
    fovCircle.Position = Vector2.new(Mouse.X, Mouse.Y + 36)
    fovCircle.Radius = aimbotFOV
    fovCircle.Visible = aiming
end)

RunService.RenderStepped:Connect(function()
    if aiming then
        local target = getClosestTarget()
        if target and target.Character and target.Character:FindFirstChild("Head") then
            local cam = workspace.CurrentCamera
            local headPos = target.Character.Head.Position
            local camPos = cam.CFrame.Position
            local desiredCFrame = CFrame.new(camPos, headPos)
            cam.CFrame = cam.CFrame:Lerp(desiredCFrame, 0.15)
        end
    end
end)

-- 5. GUI com MaterialLua - Reconstruída ao trocar tema
local themes = { "Dark", "Light", "Aqua", "Jester" }
local UI, PlayerTab

local function buildUI(theme)
    if UI then UI:Destroy() end
    UI = Material.Load({ Title="Runix Hub", Style=1, SizeX=400, SizeY=400, Theme=theme or "Dark" })
    PlayerTab = UI.New({ Title="Player" })

    PlayerTab.Toggle({ Text="Ativar ESP", Callback=function(s) config.espEnabled = s updateAll() end, Enabled=config.espEnabled })
    PlayerTab.Toggle({ Text="Hitbox Expandida (inimigos)", Callback=function(s) hitboxConfig.enabled = s updateAll() end, Enabled=hitboxConfig.enabled })
    PlayerTab.Toggle({ Text="Aimbot", Callback=function(s) aiming = s end, Enabled=aiming })

    PlayerTab.Slider({ Text="Tamanho Hitbox", Min=2, Max=15, Def=hitboxConfig.size.X,
        Callback=function(v) hitboxConfig.size = Vector3.new(v,v,v) updateAll() end })

    PlayerTab.Slider({ Text="Campo de Visão Aimbot", Min=20, Max=300, Def=aimbotFOV,
        Callback=function(v) aimbotFOV = v end })

    PlayerTab.Dropdown({
        Text="Tema da UI",
        Options=themes,
        Callback=function(sel)
            buildUI(sel) -- reconstrói tudo com o novo tema
        end
    })

    UI.Init()
end

buildUI("Dark") -- inicia com tema padrão

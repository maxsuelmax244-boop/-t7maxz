--=========================================================
-- SERVIÇOS
--=========================================================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

--=========================================================
-- CONFIG
--=========================================================
local ESP_ENABLED = false
local AIMBOT_ENABLED = false
local FOV_ENABLED = false
local SPEED_ENABLED = false

local AIMBOT_FOV = 150
local AIMBOT_SMOOTHNESS = 0.45

local DEFAULT_SPEED = 16
local SPEED_VALUE = 28

--=========================================================
-- TABELAS
--=========================================================
local ESPObjects = {}
local FOVLines = {}

--=========================================================
-- SPEED
--=========================================================
local function ApplySpeed()
    local char = LocalPlayer.Character
    if not char then return end

    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return end

    hum.WalkSpeed = SPEED_ENABLED and SPEED_VALUE or DEFAULT_SPEED
end

LocalPlayer.CharacterAdded:Connect(function()
    task.wait(0.5)
    ApplySpeed()
end)

--=========================================================
-- FOV CIRCLE
--=========================================================
local FOVCircle = Drawing.new("Circle")
FOVCircle.Thickness = 2
FOVCircle.Color = Color3.fromRGB(255,255,255)
FOVCircle.Transparency = 1
FOVCircle.Filled = false
FOVCircle.Visible = false

--=========================================================
-- VISIBILIDADE (HEAD)
--=========================================================
local function IsVisible(character)
    local head = character:FindFirstChild("Head")
    if not head then return false end

    local origin = Camera.CFrame.Position
    local direction = head.Position - origin

    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Blacklist
    params.FilterDescendantsInstances = {
        LocalPlayer.Character,
        character
    }

    local result = workspace:Raycast(origin, direction, params)
    return not result
end

--=========================================================
-- ESP
--=========================================================
local function CreateESP(player)
    if player == LocalPlayer then return end

    local lines = {}
    for i = 1,4 do
        local l = Drawing.new("Line")
        l.Thickness = 2
        l.Transparency = 1
        l.Visible = false
        table.insert(lines, l)
    end
    ESPObjects[player] = lines
end

local function RemoveESP(player)
    if ESPObjects[player] then
        for _,l in ipairs(ESPObjects[player]) do
            l:Remove()
        end
        ESPObjects[player] = nil
    end
end

local function ClearESP()
    for p,_ in pairs(ESPObjects) do
        RemoveESP(p)
    end
end

--=========================================================
-- FOV LINE
--=========================================================
local function GetFOVLine(player)
    if not FOVLines[player] then
        local l = Drawing.new("Line")
        l.Thickness = 1.5
        l.Transparency = 1
        l.Visible = false
        FOVLines[player] = l
    end
    return FOVLines[player]
end

local function RemoveFOVLine(player)
    if FOVLines[player] then
        FOVLines[player]:Remove()
        FOVLines[player] = nil
    end
end

--=========================================================
-- AIMBOT TARGET
--=========================================================
local function GetClosestTarget()
    local closest = nil
    local shortest = AIMBOT_FOV

    local center = Vector2.new(
        Camera.ViewportSize.X / 2,
        Camera.ViewportSize.Y / 2
    )

    for _,p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character then
            local hum = p.Character:FindFirstChild("Humanoid")
            local head = p.Character:FindFirstChild("Head")

            if hum and head and hum.Health > 0 then
                local pos,on = Camera:WorldToViewportPoint(head.Position)
                if on and pos.Z > 0 then
                    local dist = (Vector2.new(pos.X,pos.Y) - center).Magnitude
                    if dist <= AIMBOT_FOV and IsVisible(p.Character) then
                        if dist < shortest then
                            shortest = dist
                            closest = p
                        end
                    end
                end
            end
        end
    end

    return closest
end

--=========================================================
-- LOOP PRINCIPAL
--=========================================================
RunService.RenderStepped:Connect(function(dt)
    local center = Vector2.new(
        Camera.ViewportSize.X/2,
        Camera.ViewportSize.Y/2
    )

    --================ ESP =================
    if ESP_ENABLED then
        for _,p in ipairs(Players:GetPlayers()) do
            if p ~= LocalPlayer and p.Character then
                local char = p.Character
                local hum = char:FindFirstChild("Humanoid")
                local root = char:FindFirstChild("HumanoidRootPart")

                if hum and root and hum.Health > 0 then
                    if not ESPObjects[p] then CreateESP(p) end

                    local color = IsVisible(char)
                        and Color3.fromRGB(0,255,0)
                        or Color3.fromRGB(255,0,0)

                    local cam = Camera.CFrame
                    local r,u = cam.RightVector, cam.UpVector
                    local w,h = 4,6
                    local c = root.Position

                    local corners3D = {
                        c + (-r*w/2)+( u*h/2),
                        c + ( r*w/2)+( u*h/2),
                        c + ( r*w/2)+(-u*h/2),
                        c + (-r*w/2)+(-u*h/2)
                    }

                    local corners2D = {}
                    local ok = true
                    for i,v in ipairs(corners3D) do
                        local p2,on = Camera:WorldToViewportPoint(v)
                        if not on or p2.Z <= 0 then
                            ok = false
                            break
                        end
                        corners2D[i] = Vector2.new(p2.X,p2.Y)
                    end

                    for i=1,4 do
                        local l = ESPObjects[p][i]
                        l.Visible = ok
                        if ok then
                            l.From = corners2D[i]
                            l.To = corners2D[i%4+1]
                            l.Color = color
                        end
                    end
                else
                    RemoveESP(p)
                end
            end
        end
    else
        ClearESP()
    end

    --================ FOV =================
    if FOV_ENABLED then
        FOVCircle.Position = center
        FOVCircle.Radius = AIMBOT_FOV
        FOVCircle.Visible = true

        for _,p in ipairs(Players:GetPlayers()) do
            if p ~= LocalPlayer and p.Character then
                local hum = p.Character:FindFirstChild("Humanoid")
                local head = p.Character:FindFirstChild("Head")

                if hum and head and hum.Health > 0 then
                    local pos,on = Camera:WorldToViewportPoint(head.Position)
                    if on and pos.Z > 0 then
                        local line = GetFOVLine(p)
                        line.From = center
                        line.To = Vector2.new(pos.X,pos.Y)
                        line.Color = IsVisible(p.Character)
                            and Color3.fromRGB(0,255,0)
                            or Color3.fromRGB(255,0,0)
                        line.Visible = true
                    else
                        RemoveFOVLine(p)
                    end
                else
                    RemoveFOVLine(p)
                end
            end
        end
    else
        FOVCircle.Visible = false
        for p,_ in pairs(FOVLines) do
            RemoveFOVLine(p)
        end
    end

    --================ AIMBOT =================
    if AIMBOT_ENABLED then
        local target = GetClosestTarget()
        if target and target.Character then
            local head = target.Character:FindFirstChild("Head")
            if head then
                local cf = CFrame.new(Camera.CFrame.Position, head.Position)
                local smooth = math.clamp(AIMBOT_SMOOTHNESS * dt * 60, 0, 1)
                Camera.CFrame = Camera.CFrame:Lerp(cf, smooth)
            end
        end
    end

    --================ SPEED =================
    ApplySpeed()
end)

Players.PlayerRemoving:Connect(function(p)
    RemoveESP(p)
    RemoveFOVLine(p)
end)

--=========================================================
-- GUI
--=========================================================
local gui = Instance.new("ScreenGui", LocalPlayer.PlayerGui)
gui.ResetOnSpawn = false

local frame = Instance.new("Frame", gui)
frame.Size = UDim2.fromOffset(260,260)
frame.Position = UDim2.fromScale(0.5,0.5)
frame.AnchorPoint = Vector2.new(0.5,0.5)
frame.BackgroundColor3 = Color3.fromRGB(18,18,18)
frame.Active = true
frame.Draggable = true
Instance.new("UICorner", frame).CornerRadius = UDim.new(0,12)

local title = Instance.new("TextLabel", frame)
title.Size = UDim2.new(1,0,0,40)
title.BackgroundTransparency = 1
title.Text = "🎯 TTK: @t7maxz"
title.TextColor3 = Color3.fromRGB(255,255,255)
title.Font = Enum.Font.GothamBold
title.TextSize = 18

local minimizeBtn = Instance.new("TextButton", frame)
minimizeBtn.Size = UDim2.fromOffset(32,32)
minimizeBtn.Position = UDim2.new(1,-40,0,4)
minimizeBtn.Text = "_"
minimizeBtn.Font = Enum.Font.GothamBold
minimizeBtn.TextSize = 20
minimizeBtn.TextColor3 = Color3.fromRGB(255,255,255)
minimizeBtn.BackgroundColor3 = Color3.fromRGB(35,35,35)
Instance.new("UICorner", minimizeBtn).CornerRadius = UDim.new(1,0)

local container = Instance.new("Frame", frame)
container.Size = UDim2.new(1,0,1,-50)
container.Position = UDim2.new(0,0,0,50)
container.BackgroundTransparency = 1

local layout = Instance.new("UIListLayout", container)
layout.Padding = UDim.new(0,10)
layout.HorizontalAlignment = Enum.HorizontalAlignment.Center

local function ToggleButton(text, callback)
    local btn = Instance.new("TextButton", container)
    btn.Size = UDim2.fromOffset(220,40)
    btn.BackgroundColor3 = Color3.fromRGB(35,35,35)
    btn.TextColor3 = Color3.fromRGB(200,200,200)
    btn.Font = Enum.Font.Gotham
    btn.TextSize = 15
    btn.Text = text .. ": OFF"
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0,10)

    local on = false
    btn.MouseButton1Click:Connect(function()
        on = not on
        callback(on)
        btn.Text = text .. (on and ": ON" or ": OFF")
        btn.BackgroundColor3 = on and Color3.fromRGB(0,120,255)
            or Color3.fromRGB(35,35,35)
    end)
end

ToggleButton("ESP", function(v) ESP_ENABLED = v end)
ToggleButton("AIMBOT", function(v) AIMBOT_ENABLED = v end)
ToggleButton("FOV", function(v) FOV_ENABLED = v end)
ToggleButton("SPEED", function(v)
    SPEED_ENABLED = v
    ApplySpeed()
end)

local openBtn = Instance.new("TextButton", gui)
openBtn.Size = UDim2.fromOffset(120,40)
openBtn.Position = frame.Position
openBtn.AnchorPoint = Vector2.new(0.5,0.5)
openBtn.Text = "ABRIR"
openBtn.Font = Enum.Font.GothamBold
openBtn.TextSize = 16
openBtn.TextColor3 = Color3.fromRGB(255,255,255)
openBtn.BackgroundColor3 = Color3.fromRGB(0,120,255)
openBtn.Visible = false
openBtn.Active = true
openBtn.Draggable = true
Instance.new("UICorner", openBtn).CornerRadius = UDim.new(0,12)

minimizeBtn.MouseButton1Click:Connect(function()
    frame.Visible = false
    openBtn.Visible = true
end)

openBtn.MouseButton1Click:Connect(function()
    frame.Visible = true
    openBtn.Visible = false
end)

--=========================================================
-- MENSAGEM FIXA (TIKTOK)
--=========================================================
local aviso = Instance.new("TextLabel", gui)
aviso.Size = UDim2.fromOffset(360,40)
aviso.Position = UDim2.new(0,10,1,-50)
aviso.BackgroundColor3 = Color3.fromRGB(0,0,0)
aviso.BackgroundTransparency = 0.3
aviso.Text = "ME SEGUE NO TIKTOK: @t7maxz"
aviso.TextColor3 = Color3.fromRGB(255,255,255)
aviso.Font = Enum.Font.GothamBold
aviso.TextSize = 16
aviso.TextXAlignment = Enum.TextXAlignment.Left
aviso.ZIndex = 10

local pad = Instance.new("UIPadding", aviso)
pad.PaddingLeft = UDim.new(0,12)

Instance.new("UICorner", aviso).CornerRadius = UDim.new(0,10)

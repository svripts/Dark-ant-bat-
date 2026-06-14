-- RAINY ANTI BAT
-- Features: Anti Bat (makes you impossible to hit), Infinite Jump (Manual/Hold), keybinds

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local LP = Players.LocalPlayer

-- ============================================================
-- SETTINGS
-- ============================================================
local antiBatEnabled = true
local infJumpEnabled = true
local infJumpMode = "Hold"   -- "Manual" or "Hold"
local antiBatKey = Enum.KeyCode.K
local jumpKey = Enum.KeyCode.J

-- ============================================================
-- CORE: ANTI BAT (hyper jitter + auto dodge)
-- ============================================================
local antiBatConn = nil
local dodgeConn = nil
local lastPos = nil

local function startAntiBat()
    if antiBatConn then antiBatConn:Disconnect() end
    if dodgeConn then dodgeConn:Disconnect() end

    -- Main jitter: random velocity every frame
    antiBatConn = RunService.Heartbeat:Connect(function()
        if not antiBatEnabled then return end
        local char = LP.Character
        if not char then return end
        local root = char:FindFirstChild("HumanoidRootPart")
        if not root then return end

        -- random horizontal velocity (80-120 studs/s)
        local angle = math.random() * math.pi * 2
        local speed = math.random(80, 120)
        local x = math.cos(angle) * speed
        local z = math.sin(angle) * speed
        local y = root.Velocity.Y
        root.Velocity = Vector3.new(x, y, z)

        -- also push away from nearest enemy (auto dodge)
        local closest, minDist = nil, 20
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LP and p.Character then
                local enemyRoot = p.Character:FindFirstChild("HumanoidRootPart")
                if enemyRoot then
                    local dist = (enemyRoot.Position - root.Position).Magnitude
                    if dist < minDist then
                        minDist = dist
                        closest = enemyRoot
                    end
                end
            end
        end
        if closest then
            local dir = (root.Position - closest.Position).Unit
            root.Velocity = dir * 120 + Vector3.new(0, root.Velocity.Y, 0)
        end
    end)

    -- Anti ragdoll: prevent being stunned
    dodgeConn = RunService.Heartbeat:Connect(function()
        if not antiBatEnabled then return end
        local char = LP.Character
        if not char then return end
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then
            local state = hum:GetState()
            if state == Enum.HumanoidStateType.Physics or
               state == Enum.HumanoidStateType.Ragdoll or
               state == Enum.HumanoidStateType.FallingDown then
                hum:ChangeState(Enum.HumanoidStateType.Running)
                hum.PlatformStand = false
                local root = char:FindFirstChild("HumanoidRootPart")
                if root then
                    root.Velocity = Vector3.zero
                    root.RotVelocity = Vector3.zero
                end
            end
        end
    end)
end

local function stopAntiBat()
    if antiBatConn then antiBatConn:Disconnect(); antiBatConn = nil end
    if dodgeConn then dodgeConn:Disconnect(); dodgeConn = nil end
end

-- ============================================================
-- INFINITE JUMP
-- ============================================================
local manualJumpPressed = false
local holdJumpActive = false

local function applyJump()
    if not infJumpEnabled then return end
    local char = LP.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if root then
        root.Velocity = Vector3.new(root.Velocity.X, 55, root.Velocity.Z)
    end
end

-- Manual mode: on JumpRequest (pressing jump key)
UserInputService.JumpRequest:Connect(function()
    if infJumpEnabled and infJumpMode == "Manual" then
        applyJump()
    end
end)

-- Hold mode: while jump key is held
UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.KeyCode == jumpKey then
        if infJumpEnabled and infJumpMode == "Hold" then
            holdJumpActive = true
            applyJump()
            task.spawn(function()
                while holdJumpActive and infJumpEnabled and infJumpMode == "Hold" do
                    applyJump()
                    task.wait(0.05)
                end
            end)
        end
    end
end)
UserInputService.InputEnded:Connect(function(input)
    if input.KeyCode == jumpKey then
        holdJumpActive = false
    end
end)

-- also support space bar for manual mode (optional)
UserInputService.JumpRequest:Connect(function()
    if infJumpEnabled and infJumpMode == "Manual" then
        applyJump()
    end
end)

-- ============================================================
-- KEYBIND HANDLER
-- ============================================================
UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.KeyCode == antiBatKey then
        antiBatEnabled = not antiBatEnabled
        if antiBatEnabled then
            startAntiBat()
        else
            stopAntiBat()
        end
        -- update GUI toggle text
        if guiToggle then guiToggle.Text = antiBatEnabled and "ANTI BAT • ENABLED" or "ANTI BAT • DISABLED" end
        if disableBtn then disableBtn.Text = antiBatEnabled and "DISABLE ANTI BAT" or "ENABLE ANTI BAT" end
    end
    if input.KeyCode == jumpKey then
        if infJumpMode == "Manual" then
            applyJump()
        end
    end
end)

-- ============================================================
-- GUI
-- ============================================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "RainyAntiBat"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true
screenGui.Parent = LP:WaitForChild("PlayerGui")

local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0, 260, 0, 220)
mainFrame.Position = UDim2.new(0.5, -130, 0.5, -110)
mainFrame.AnchorPoint = Vector2.new(0.5, 0.5)
mainFrame.BackgroundColor3 = Color3.fromRGB(12, 12, 20)
mainFrame.BorderSizePixel = 0
mainFrame.Active = true
mainFrame.Draggable = true
mainFrame.Parent = screenGui
Instance.new("UICorner", mainFrame).CornerRadius = UDim.new(0, 14)
Instance.new("UIStroke", mainFrame).Color = Color3.fromRGB(80, 150, 255)

-- Title
local title = Instance.new("TextLabel", mainFrame)
title.Size = UDim2.new(1, 0, 0, 40)
title.Position = UDim2.new(0, 0, 0, 0)
title.BackgroundTransparency = 1
title.Text = "RAINY ANTI BAT"
title.TextColor3 = Color3.fromRGB(80, 150, 255)
title.Font = Enum.Font.GothamBlack
title.TextSize = 18

-- Anti Bat status toggle (display)
local guiToggle = Instance.new("TextLabel", mainFrame)
guiToggle.Size = UDim2.new(1, -20, 0, 30)
guiToggle.Position = UDim2.new(0, 10, 0, 45)
guiToggle.BackgroundTransparency = 1
guiToggle.Text = antiBatEnabled and "ANTI BAT • ENABLED" or "ANTI BAT • DISABLED"
guiToggle.TextColor3 = Color3.fromRGB(255, 200, 100)
guiToggle.Font = Enum.Font.GothamBold
guiToggle.TextSize = 14
guiToggle.TextXAlignment = Enum.TextXAlignment.Center

-- Disable/Enable button
local disableBtn = Instance.new("TextButton", mainFrame)
disableBtn.Size = UDim2.new(0.8, 0, 0, 36)
disableBtn.Position = UDim2.new(0.1, 0, 0, 80)
disableBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 50)
disableBtn.Text = antiBatEnabled and "DISABLE ANTI BAT" or "ENABLE ANTI BAT"
disableBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
disableBtn.Font = Enum.Font.GothamBold
disableBtn.TextSize = 13
Instance.new("UICorner", disableBtn).CornerRadius = UDim.new(0, 8)
Instance.new("UIStroke", disableBtn).Color = Color3.fromRGB(80, 150, 255)
disableBtn.MouseButton1Click:Connect(function()
    antiBatEnabled = not antiBatEnabled
    if antiBatEnabled then
        startAntiBat()
    else
        stopAntiBat()
    end
    guiToggle.Text = antiBatEnabled and "ANTI BAT • ENABLED" or "ANTI BAT • DISABLED"
    disableBtn.Text = antiBatEnabled and "DISABLE ANTI BAT" or "ENABLE ANTI BAT"
end)

-- Infinite Jump label
local infJumpLabel = Instance.new("TextLabel", mainFrame)
infJumpLabel.Size = UDim2.new(0.6, 0, 0, 30)
infJumpLabel.Position = UDim2.new(0.05, 0, 0, 125)
infJumpLabel.BackgroundTransparency = 1
infJumpLabel.Text = "Infinite Jump"
infJumpLabel.TextColor3 = Color3.fromRGB(220, 220, 220)
infJumpLabel.Font = Enum.Font.GothamBold
infJumpLabel.TextSize = 13
infJumpLabel.TextXAlignment = Enum.TextXAlignment.Left

-- Manual/Hold toggle
local modeBtn = Instance.new("TextButton", mainFrame)
modeBtn.Size = UDim2.new(0.3, 0, 0, 30)
modeBtn.Position = UDim2.new(0.65, 0, 0, 125)
modeBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 60)
modeBtn.Text = infJumpMode
modeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
modeBtn.Font = Enum.Font.GothamBold
modeBtn.TextSize = 12
Instance.new("UICorner", modeBtn).CornerRadius = UDim.new(0, 6)
modeBtn.MouseButton1Click:Connect(function()
    infJumpMode = (infJumpMode == "Manual") and "Hold" or "Manual"
    modeBtn.Text = infJumpMode
end)

-- Anti Bat Key display
local antiKeyLabel = Instance.new("TextLabel", mainFrame)
antiKeyLabel.Size = UDim2.new(0.55, 0, 0, 28)
antiKeyLabel.Position = UDim2.new(0.05, 0, 0, 165)
antiKeyLabel.BackgroundTransparency = 1
antiKeyLabel.Text = "Anti Bat Key:"
antiKeyLabel.TextColor3 = Color3.fromRGB(180, 180, 200)
antiKeyLabel.Font = Enum.Font.GothamBold
antiKeyLabel.TextSize = 12
antiKeyLabel.TextXAlignment = Enum.TextXAlignment.Left

local antiKeyBtn = Instance.new("TextButton", mainFrame)
antiKeyBtn.Size = UDim2.new(0.25, 0, 0, 28)
antiKeyBtn.Position = UDim2.new(0.68, 0, 0, 165)
antiKeyBtn.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
antiKeyBtn.Text = tostring(antiBatKey.Name)
antiKeyBtn.TextColor3 = Color3.fromRGB(80, 150, 255)
antiKeyBtn.Font = Enum.Font.GothamBold
antiKeyBtn.TextSize = 12
Instance.new("UICorner", antiKeyBtn).CornerRadius = UDim.new(0, 6)
local listeningAnti = false
antiKeyBtn.MouseButton1Click:Connect(function()
    if listeningAnti then return end
    listeningAnti = true
    antiKeyBtn.Text = "..."
    local conn
    conn = UserInputService.InputBegan:Connect(function(inp)
        if not listeningAnti then return end
        if inp.UserInputType == Enum.UserInputType.Keyboard then
            antiBatKey = inp.KeyCode
            antiKeyBtn.Text = tostring(antiBatKey.Name)
            listeningAnti = false
            conn:Disconnect()
        end
    end)
end)

-- Jump Key display
local jumpKeyLabel = Instance.new("TextLabel", mainFrame)
jumpKeyLabel.Size = UDim2.new(0.55, 0, 0, 28)
jumpKeyLabel.Position = UDim2.new(0.05, 0, 0, 195)
jumpKeyLabel.BackgroundTransparency = 1
jumpKeyLabel.Text = "Jump Key:"
jumpKeyLabel.TextColor3 = Color3.fromRGB(180, 180, 200)
jumpKeyLabel.Font = Enum.Font.GothamBold
jumpKeyLabel.TextSize = 12
jumpKeyLabel.TextXAlignment = Enum.TextXAlignment.Left

local jumpKeyBtn = Instance.new("TextButton", mainFrame)
jumpKeyBtn.Size = UDim2.new(0.25, 0, 0, 28)
jumpKeyBtn.Position = UDim2.new(0.68, 0, 0, 195)
jumpKeyBtn.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
jumpKeyBtn.Text = tostring(jumpKey.Name)
jumpKeyBtn.TextColor3 = Color3.fromRGB(80, 150, 255)
jumpKeyBtn.Font = Enum.Font.GothamBold
jumpKeyBtn.TextSize = 12
Instance.new("UICorner", jumpKeyBtn).CornerRadius = UDim.new(0, 6)
local listeningJump = false
jumpKeyBtn.MouseButton1Click:Connect(function()
    if listeningJump then return end
    listeningJump = true
    jumpKeyBtn.Text = "..."
    local conn
    conn = UserInputService.InputBegan:Connect(function(inp)
        if not listeningJump then return end
        if inp.UserInputType == Enum.UserInputType.Keyboard then
            jumpKey = inp.KeyCode
            jumpKeyBtn.Text = tostring(jumpKey.Name)
            listeningJump = false
            conn:Disconnect()
        end
    end)
end)

-- Start Anti Bat by default
startAntiBat()

print("Rainy Anti Bat loaded | discord.gg/rainyhub")

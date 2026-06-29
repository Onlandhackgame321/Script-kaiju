-- [[ KAIJU ALPHA - ULTIMATE SILENT BEAM FREECAM (NEXIT HUB REPLICATED) ]]
if game.CoreGui:FindFirstChild("KaijuChainBeamHub") then
    game.CoreGui.KaijuChainBeamHub:Destroy()
end

local ScreenGui = Instance.new("ScreenGui")
local MainFrame = Instance.new("Frame")
local Title = Instance.new("TextLabel")
local ToggleButton = Instance.new("TextButton")
local CloseOpenButton = Instance.new("TextButton")
local UICorner = Instance.new("UICorner")
local UICorner2 = Instance.new("UICorner")
local UICorner3 = Instance.new("UICorner")

ScreenGui.Name = "KaijuChainBeamHub"
ScreenGui.Parent = game:GetService("CoreGui")
ScreenGui.ResetOnSpawn = false

-- [[ NÚT BẤM ẨN/HIỆN GUI CHÍNH - GIỮA TRÊN CÙNG MÀN HÌNH ]]
CloseOpenButton.Name = "CloseOpenButton"
CloseOpenButton.Parent = ScreenGui
CloseOpenButton.BackgroundColor3 = Color3.fromRGB(35, 30, 45)
CloseOpenButton.BackgroundTransparency = 0.2
CloseOpenButton.Position = UDim2.new(0.5, -45, 0, 8) 
CloseOpenButton.Size = UDim2.new(0, 90, 0, 28)
CloseOpenButton.Font = Enum.Font.SourceSansBold
CloseOpenButton.Text = "Nexit Hub : ON"
CloseOpenButton.TextColor3 = Color3.fromRGB(0, 255, 150)
CloseOpenButton.TextSize = 13
UICorner3.CornerRadius = UDim.new(0, 6)
UICorner3.Parent = CloseOpenButton

-- [[ BẢNG MENU CHÍNH ]]
MainFrame.Name = "MainFrame"
MainFrame.Parent = ScreenGui
MainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
MainFrame.Position = UDim2.new(0.1, 0, 0.15, 0)
MainFrame.Size = UDim2.new(0, 220, 0, 130)
MainFrame.Active = true
MainFrame.Draggable = true
UICorner.Parent = MainFrame

Title.Name = "Title"
Title.Parent = MainFrame
Title.BackgroundTransparency = 1
Title.Size = UDim2.new(1, 0, 0.3, 0)
Title.Font = Enum.Font.SourceSansBold
Title.Text = "KAIJU SILENT BEAM"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.TextSize = 15

ToggleButton.Name = "ToggleButton"
ToggleButton.Parent = MainFrame
ToggleButton.BackgroundColor3 = Color3.fromRGB(220, 50, 50)
ToggleButton.Position = UDim2.new(0.1, 0, 0.45, 0)
ToggleButton.Size = UDim2.new(0.8, 0, 0.4, 0)
ToggleButton.Font = Enum.Font.SourceSansBold
ToggleButton.Text = "Auto Beam Players: OFF"
ToggleButton.TextColor3 = Color3.fromRGB(255, 255, 255)
ToggleButton.TextSize = 14
UICorner2.Parent = ToggleButton

-- [[ KẾT NỐI SỰ KIỆN ẨN / HIỆN GUI ]]
CloseOpenButton.MouseButton1Click:Connect(function()
    MainFrame.Visible = not MainFrame.Visible
    if MainFrame.Visible then
        CloseOpenButton.Text = "Nexit Hub : ON"
        CloseOpenButton.TextColor3 = Color3.fromRGB(0, 255, 150)
    else
        CloseOpenButton.Text = "Nexit Hub : OFF"
        CloseOpenButton.TextColor3 = Color3.fromRGB(255, 100, 100)
    end
end)

-- [[ LOGIC SILENT AIM & CORE BYPASS ]]
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()
local Camera = workspace.CurrentCamera

local isScriptActive = false
local targetConnection = nil
local currentTarget = nil
local lastSwitchTime = 0
local SWITCH_TIME = 0.5
local MAX_LOCK_DISTANCE = 600
local hitTargets = {}

-- TOÀN CỤC HOOK: Bẻ hướng cả Mouse lẫn CFrame Camera ngầm khi game check hướng bắn
local indexHook
indexHook = hookmetamethod(game, "__index", function(self, key)
    if isScriptActive and currentTarget and currentTarget:IsA("BasePart") then
        -- Đánh lừa hướng trỏ chuột trái
        if self == Mouse and (key == "Hit" or key == "Target") then
            if key == "Hit" then return currentTarget.CFrame end
            if key == "Target" then return currentTarget end
        end
        -- Đánh lừa góc nhìn Camera gốc (Game lấy góc này để khè Beam)
        if self == Camera and key == "CFrame" then
            return CFrame.lookAt(hookMouse(self, "CFrame").Position, currentTarget.Position)
        end
    end
    return indexHook(self, key)
end)

-- Hàm quét tìm đối thủ gần nhất
local function findNextTarget()
    local myChar = LocalPlayer.Character
    local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if not myRoot then return nil end
    
    local potentialTargets = {}
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local enemyChar = player.Character
            local enemyRoot = enemyChar:FindFirstChild("HumanoidRootPart")
            local enemyHumanoid = enemyChar:FindFirstChildOfClass("Humanoid")
            
            if enemyRoot and enemyHumanoid and enemyHumanoid.Health > 0 then
                local distance = (enemyRoot.Position - myRoot.Position).Magnitude
                if distance <= MAX_LOCK_DISTANCE then
                    if not hitTargets[player.Name] then
                        table.insert(potentialTargets, {root = enemyRoot, dist = distance, name = player.Name})
                    end
                end
            end
        end
    end
    
    if #potentialTargets == 0 and next(hitTargets) ~= nil then
        hitTargets = {}
        return findNextTarget()
    end
    
    table.sort(potentialTargets, function(a, b) return a.dist < b.dist end)
    if #potentialTargets > 0 then
        return potentialTargets[1].root, potentialTargets[1].name
    end
    return nil, nil
end

-- Vòng lặp quét mục tiêu ngầm (Không tác động gì tới Camera/Nhân vật vật lý)
local function updateTargetLogic()
    local currentTime = os.clock()
    
    if not currentTarget or (currentTime - lastSwitchTime >= SWITCH_TIME) or (currentTarget.Parent and currentTarget.Parent:FindFirstChildOfClass("Humanoid") and currentTarget.Parent:FindFirstChildOfClass("Humanoid").Health <= 0) then
        if currentTarget and currentTarget.Parent then
            local pName = Players:GetPlayerFromCharacter(currentTarget.Parent)
            if pName then hitTargets[pName.Name] = true end
        end
        
        local nextRoot, nextName = findNextTarget()
        if nextRoot then
            currentTarget = nextRoot
            lastSwitchTime = currentTime
        else
            currentTarget = nil
        end
    end
end

-- Cơ chế kích hoạt skill tự động
local function autoBeamExecution()
    if isScriptActive and currentTarget then
        VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.Two, false, game)
        task.wait(0.01)
        VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.Two, false, game)
        
        pcall(function()
            VirtualInputManager:SendMouseButtonEvent(0, 0, 0, true, game, 1)
            task.wait(0.4)
            VirtualInputManager:SendMouseButtonEvent(0, 0, 0, false, game, 1)
        end)
    end
end

-- Trạng thái Bật/Tắt hệ thống
ToggleButton.MouseButton1Click:Connect(function()
    isScriptActive = not isScriptActive
    
    if isScriptActive then
        ToggleButton.Text = "Auto Beam Players: ON"
        ToggleButton.BackgroundColor3 = Color3.fromRGB(50, 180, 50)
        
        hitTargets = {}
        currentTarget = nil
        lastSwitchTime = 0
        
        targetConnection = RunService.Heartbeat:Connect(updateTargetLogic)
        
        task.spawn(function()
            while isScriptActive do
                autoBeamExecution()
                task.wait(0.05)
            end
        end)
    else
        ToggleButton.Text = "Auto Beam Players: OFF"
        ToggleButton.BackgroundColor3 = Color3.fromRGB(220, 50, 50)
        
        if targetConnection then
            targetConnection:Disconnect()
            targetConnection = nil
        end
        isScriptActive = false
        currentTarget = nil
        hitTargets = {}
    end
end)

LocalPlayer.CharacterRemoving:Connect(function()
    if isScriptActive then
        isScriptActive = false
        ToggleButton.Text = "Auto Beam Players: OFF"
        ToggleButton.BackgroundColor3 = Color3.fromRGB(220, 50, 50)
        if targetConnection then targetConnection:Disconnect() end
        currentTarget = nil
        hitTargets = {}
    end
end)

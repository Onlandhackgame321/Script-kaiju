-- [[ KAIJU ALPHA - FINAL PERFECT & STABLE SYSTEM FOR DELTA ]]
if game.CoreGui:FindFirstChild("KaijuChainBeamHub") then
    game.CoreGui.KaijuChainBeamHub:Destroy()
end

local ScreenGui = Instance.new("ScreenGui")
local MainFrame = Instance.new("Frame")
local Title = Instance.new("TextLabel")
local ToggleButton = Instance.new("TextButton")
local UICorner = Instance.new("UICorner")
local UICorner2 = Instance.new("UICorner")

ScreenGui.Name = "KaijuChainBeamHub"
ScreenGui.Parent = game:GetService("CoreGui")
ScreenGui.ResetOnSpawn = false

MainFrame.Name = "MainFrame"
MainFrame.Parent = ScreenGui
MainFrame.BackgroundColor3 = Color3.fromRGB(30, 25, 35)
MainFrame.Position = UDim2.new(0.1, 0, 0.1, 0)
MainFrame.Size = UDim2.new(0, 220, 0, 130)
MainFrame.Active = true
MainFrame.Draggable = true
UICorner.Parent = MainFrame

Title.Name = "Title"
Title.Parent = MainFrame
Title.BackgroundTransparency = 1
Title.Size = UDim2.new(1, 0, 0.3, 0)
Title.Font = Enum.Font.SourceSansBold
Title.Text = "KAIJU TRUE AIM (0.5s)"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.TextSize = 15

ToggleButton.Name = "ToggleButton"
ToggleButton.Parent = MainFrame
ToggleButton.BackgroundColor3 = Color3.fromRGB(220, 50, 50)
ToggleButton.Position = UDim2.new(0.1, 0, 0.45, 0)
ToggleButton.Size = UDim2.new(0.8, 0, 0.4, 0)
ToggleButton.Font = Enum.Font.SourceSansBold
ToggleButton.Text = "Chain Skill: OFF"
ToggleButton.TextColor3 = Color3.fromRGB(255, 255, 255)
ToggleButton.TextSize = 16
UICorner2.Parent = ToggleButton

-- [[ SKILL CASTING & MOUSE LOCK LOGIC ]]
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()
local Camera = workspace.CurrentCamera

local isScriptActive = false
local renderConnection = nil
local MAX_LOCK_DISTANCE = 500
local SWITCH_TIME = 0.5
local currentTarget = nil
local lastSwitchTime = 0
local hitTargets = {}

-- GLOBAL MOUSE HOOK: Chạy duy nhất 1 lần để bẻ hướng chuột cực kỳ ổn định
local hookMouse
hookMouse = hookmetamethod(game, "__index", function(self, key)
    if self == Mouse and (key == "Hit" or key == "Target") and isScriptActive and currentTarget then
        if key == "Hit" then
            return currentTarget.CFrame
        elseif key == "Target" then
            return currentTarget
        end
    end
    return hookMouse(self, key)
end)

-- Function to find nearby valid targets
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

-- Function to cast skill and safely simulate real input
local function castKaijuSkillAtTarget()
    -- Step 1: Chọn chiêu thức số 2
    VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.Two, false, game)
    task.wait(0.02)
    VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.Two, false, game)
    task.wait(0.03)
    
    -- Step 2: Nhấn giữ chuột ảo (Bẻ hướng tự động xử lý bởi Hook ở trên)
    pcall(function()
        VirtualInputManager:SendMouseButtonEvent(0, 0, 0, true, game, 1)
        task.wait(0.4) -- Khè tia liên tục trong 0.4 giây
        VirtualInputManager:SendMouseButtonEvent(0, 0, 0, false, game, 1)
    end)
end

-- Update target logic and lock camera orientation
local function updateSkillTarget()
    local myChar = LocalPlayer.Character
    local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if not myRoot then return end
    
    local currentTime = os.clock()
    
    -- Switch target every 0.5 seconds
    if not currentTarget or (currentTime - lastSwitchTime >= SWITCH_TIME) or (currentTarget.Parent and currentTarget.Parent:FindFirstChildOfClass("Humanoid") and currentTarget.Parent:FindFirstChildOfClass("Humanoid").Health <= 0) then
        if currentTarget and currentTarget.Parent then
            local pName = Players:GetPlayerFromCharacter(currentTarget.Parent)
            if pName then hitTargets[pName.Name] = true end
        end
        
        local nextRoot, nextName = findNextTarget()
        if nextRoot then
            currentTarget = nextRoot
            lastSwitchTime = currentTime
            
            -- Chạy xả chiêu trên luồng độc lập không gây khựng màn hình
            task.spawn(castKaijuSkillAtTarget)
        else
            currentTarget = nil
        end
    end
    
    -- Always lock Camera and Character towards the target
    if currentTarget and currentTarget.Parent then
        local targetPos = currentTarget.Position
        Camera.CFrame = CFrame.lookAt(Camera.CFrame.Position, targetPos)
        myRoot.CFrame = CFrame.lookAt(myRoot.Position, Vector3.new(targetPos.X, myRoot.Position.Y, targetPos.Z))
    end
end

-- Toggle System Connection
ToggleButton.MouseButton1Click:Connect(function()
    isScriptActive = not isScriptActive
    
    if isScriptActive then
        ToggleButton.Text = "Chain Skill: ON"
        ToggleButton.BackgroundColor3 = Color3.fromRGB(50, 180, 50)
        
        hitTargets = {}
        currentTarget = nil
        lastSwitchTime = 0
        
        renderConnection = RunService.RenderStepped:Connect(updateSkillTarget)
    else
        ToggleButton.Text = "Chain Skill: OFF"
        ToggleButton.BackgroundColor3 = Color3.fromRGB(220, 50, 50)
        
        if renderConnection then
            renderConnection:Disconnect()
            renderConnection = nil
        end
        currentTarget = nil
        hitTargets = {}
    end
end)

LocalPlayer.CharacterRemoving:Connect(function()
    if isScriptActive then
        isScriptActive = false
        ToggleButton.Text = "Chain Skill: OFF"
        ToggleButton.BackgroundColor3 = Color3.fromRGB(220, 50, 50)
        if renderConnection then renderConnection:Disconnect() end
        currentTarget = nil
        hitTargets = {}
    end
end)

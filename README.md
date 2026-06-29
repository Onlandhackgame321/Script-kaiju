-- [[ KAIJU ALPHA - ULTIMATE SILENT BEAM (PREDICTION VERSION) ]]

-- [CONFIG]
local PredictionValue = 0.25 -- Tăng số này nếu Beam bị bắn sau lưng đối thủ
local Smoothing = 0.1 

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local Mouse = LocalPlayer:GetMouse()

local isScriptActive = false
local currentTarget = nil

-- [[ BẺ HƯỚNG BẰNG HOOKMETAMETHOD (AN TOÀN HƠN) ]]
local oldIndex
oldIndex = hookmetamethod(game, "__index", function(self, key)
    if isScriptActive and currentTarget and currentTarget:IsA("BasePart") then
        local predictedPos = currentTarget.Position + (currentTarget.Velocity * PredictionValue)
        
        -- Ép chuột ngầm (Silent Aim)
        if self == Mouse and (key == "Hit" or key == "Target") then
            if key == "Hit" then return CFrame.new(predictedPos) end
            if key == "Target" then return currentTarget end
        end
        
        -- Ép Camera ngầm (Silent Homing)
        if self == Camera and key == "CFrame" then
            local original = oldIndex(self, key)
            return CFrame.lookAt(original.Position, predictedPos)
        end
    end
    return oldIndex(self, key)
end)

-- [[ TỐI ƯU HÀM QUÉT MỤC TIÊU ]]
local function getClosest()
    local closest, maxDist = nil, 600
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            local root = p.Character.HumanoidRootPart
            local dist = (root.Position - LocalPlayer.Character.HumanoidRootPart.Position).Magnitude
            if dist < maxDist then
                closest = root
                maxDist = dist
            end
        end
    end
    return closest
end

-- [[ VÒNG LẶP CẬP NHẬT MỤC TIÊU ]]
RunService.RenderStepped:Connect(function()
    if isScriptActive then
        currentTarget = getClosest()
    end
end)

-- [[ PHẦN KÍCH HOẠT (ĐÃ FIX LAG INPUT) ]]
task.spawn(function()
    while true do
        if isScriptActive and currentTarget then
            -- Spam skill 2 nhẹ nhàng
            VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.Two, false, game)
            task.wait(0.05)
            VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.Two, false, game)
            
            -- Click chuột để bắn
            VirtualInputManager:SendMouseButtonEvent(0, 0, 0, true, game, 1)
            task.wait(0.3)
            VirtualInputManager:SendMouseButtonEvent(0, 0, 0, false, game, 1)
        end
        task.wait(0.1)
    end
end)

-- [PHẦN GUI GIỮ NGUYÊN NHƯ BẠN ĐÃ CÓ - CHỈ CẦN GẮN BIẾN isScriptActive VÀO NÚT TOGGLE]

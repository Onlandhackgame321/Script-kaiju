-- [[ KAIJU ALPHA - CHAIN SWITCH BEAM SYSTEM FOR DELTA ]]

-- 1. Khởi tạo Giao diện Người dùng (GUI)
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
Title.Text = "KAIJU CHAIN BEAM (0.5s)"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.TextSize = 15

ToggleButton.Name = "ToggleButton"
ToggleButton.Parent = MainFrame
ToggleButton.BackgroundColor3 = Color3.fromRGB(220, 50, 50)
ToggleButton.Position = UDim2.new(0.1, 0, 0.45, 0)
ToggleButton.Size = UDim2.new(0.8, 0, 0.4, 0)
ToggleButton.Font = Enum.Font.SourceSansBold
ToggleButton.Text = "Chain Beam: OFF"
ToggleButton.TextColor3 = Color3.fromRGB(255, 255, 255)
ToggleButton.TextSize = 16

UICorner2.Parent = ToggleButton

-- 2. Cấu hình & Logic Vòng lặp Nhảy Mục Tiêu
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer

local isBeamActive = false
local currentBeamPart = nil
local renderConnection = nil

local MAX_LOCK_DISTANCE = 300 -- Khoảng cách quét mục tiêu
local SWITCH_TIME = 0.5 -- Thời gian giữ trên mỗi mục tiêu (0.5 giây)

local currentTarget = nil
local lastSwitchTime = 0
local hitTargets = {} -- Danh sách lưu các Player đã bị bắn qua trong lượt này

-- Hàm tìm kiếm một mục tiêu hợp lệ chưa từng bị bắn trong lượt này
local function findNextTarget()
    local myChar = LocalPlayer.Character
    local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if not myRoot then return nil end

    local potentialTargets = {}

    -- Lọc ra tất cả các Player đủ điều kiện trong tầm bắn
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local enemyChar = player.Character
            local enemyRoot = enemyChar:FindFirstChild("HumanoidRootPart")
            local enemyHumanoid = enemyChar:FindFirstChildOfClass("Humanoid")

            if enemyRoot and enemyHumanoid and enemyHumanoid.Health > 0 then
                local distance = (enemyRoot.Position - myRoot.Position).Magnitude
                if distance <= MAX_LOCK_DISTANCE then
                    -- Chỉ thêm vào danh sách nếu Player này chưa bị bắn trong lượt hiện tại
                    if not hitTargets[player.Name] then
                        table.insert(potentialTargets, {root = enemyRoot, dist = distance, name = player.Name})
                    end
                end
            end
        end
    end

    -- Nếu tất cả các mục tiêu xung quanh đều đã bị bắn hết, reset lại danh sách để bắt đầu lượt quét mới
    if #potentialTargets == 0 and next(hitTargets) ~= nil then
        hitTargets = {} -- Xóa lịch sử đã bắn
        return findNextTarget() -- Quét lại từ đầu
    end

    -- Sắp xếp để ưu tiên bắn mục tiêu gần nhất trước
    table.sort(potentialTargets, function(a, b) return a.dist < b.dist end)

    if #potentialTargets > 0 then
        return potentialTargets[1].root, potentialTargets[1].name
    end

    return nil, nil
end

-- Hàm tạo Part hiệu ứng tia Beam
local function createBeamEffect()
    local beam = Instance.new("Part")
    beam.Name = "KaijuChainBeam"
    beam.Shape = Enum.PartType.Cylinder
    beam.Material = Enum.Material.Neon
    beam.BrickColor = BrickColor.new("Bright yellow") -- Đổi thành màu vàng/cam cho giống đòn sấm sét chuyển hướng
    beam.CanCollide = false
    beam.Anchored = true
    beam.Parent = workspace
    return beam
end

-- Hàm cập nhật vị trí tia Beam và xử lý nhảy mục tiêu
local function updateBeamPosition()
    local myChar = LocalPlayer.Character
    if not myChar then return end
    
    local emitter = myChar:FindFirstChild("Head") or myChar:FindFirstChild("HumanoidRootPart")
    if not emitter or not currentBeamPart then return end
    
    local currentTime = os.clock()
    
    -- Kiểm tra xem đã hết 0.5 giây chưa hoặc mục tiêu cũ đã chết/ra xa chưa
    if not currentTarget or (currentTime - lastSwitchTime >= SWITCH_TIME) or (currentTarget.Parent and currentTarget.Parent:FindFirstChildOfClass("Humanoid") and currentTarget.Parent:FindFirstChildOfClass("Humanoid").Health <= 0) then
        
        -- Nếu có mục tiêu cũ vừa hết thời gian 0.5s, đưa vào danh sách đen đã bắn
        if currentTarget and currentTarget.Parent then
            local pName = Players:GetPlayerFromCharacter(currentTarget.Parent)
            if pName then
                hitTargets[pName.Name] = true
            end
        end

        -- Tìm mục tiêu mới
        local nextRoot, nextName = findNextTarget()
        if nextRoot then
            currentTarget = nextRoot
            lastSwitchTime = currentTime
        else
            currentTarget = nil -- Không có ai xung quanh
        end
    end

    -- Xác định tọa độ bắn
    local originPos = emitter.Position
    local targetPos = nil

    if currentTarget and currentTarget.Parent and currentTarget.Parent:FindFirstChild("HumanoidRootPart") then
        targetPos = currentTarget.Position
    else
        -- Nếu không tìm thấy ai, tự động chỉ theo con trỏ chuột/tay chạm
        targetPos = LocalPlayer:GetMouse().Hit.Position
    end

    local distance = (targetPos - originPos).Magnitude
    if distance > MAX_LOCK_DISTANCE then
        distance = MAX_LOCK_DISTANCE
        targetPos = originPos + (targetPos - originPos).Unit * MAX_LOCK_DISTANCE
    end

    -- Định hình lại khối CFrame tia Beam
    currentBeamPart.Size = Vector3.new(distance, 4, 4)
    currentBeamPart.CFrame = CFrame.lookAt(originPos, targetPos) * CFrame.new(0, 0, -distance/2) * CFrame.Angles(0, math.rad(90), 0)
end

-- 3. Quản lý Sự kiện Nút bấm trên Delta
ToggleButton.MouseButton1Click:Connect(function()
    isBeamActive = not isBeamActive
    
    if isBeamActive then
        ToggleButton.Text = "Chain Beam: ON"
        ToggleButton.BackgroundColor3 = Color3.fromRGB(50, 180, 50)
        
        hitTargets = {} -- Reset danh sách đã bắn khi kích hoạt lượt mới
        currentTarget = nil
        lastSwitchTime = 0
        
        currentBeamPart = createBeamEffect()
        renderConnection = RunService.RenderStepped:Connect(updateBeamPosition)
    else
        ToggleButton.Text = "Chain Beam: OFF"
        ToggleButton.BackgroundColor3 = Color3.fromRGB(220, 50, 50)
        
        if renderConnection then
            renderConnection:Disconnect()
            renderConnection = nil
        end
        if currentBeamPart then
            currentBeamPart:Destroy()
            currentBeamPart = nil
        end
        currentTarget = nil
        hitTargets = {}
    end
end)

-- Tự động dọn dẹp khi nhân vật chết
LocalPlayer.CharacterRemoving:Connect(function()
    if isBeamActive then
        isBeamActive = false
        ToggleButton.Text = "Chain Beam: OFF"
        ToggleButton.BackgroundColor3 = Color3.fromRGB(220, 50, 50)
        if renderConnection then renderConnection:Disconnect() end
        if currentBeamPart then currentBeamPart:Destroy() end
        currentTarget = nil
        hitTargets = {}
    end
end)

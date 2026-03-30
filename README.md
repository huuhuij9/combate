-- ============================================================
-- LÓGICA DE COMBATE
-- ============================================================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local Mouse = LocalPlayer:GetMouse()
local Allies = {}

local function setCharacterTransparency(character, transparency)
    for _, descendant in pairs(character:GetDescendants()) do
        if descendant:IsA("BasePart") or descendant:IsA("Decal") then descendant.Transparency = transparency end
    end
end

local function toggleInvisibility()
    local character = LocalPlayer.Character
    if not character then return end
    if getgenv().Invisibility_Enabled then
        local root = character:FindFirstChild("HumanoidRootPart")
        if not root then return end
        local savedPosition = root.CFrame
        character:MoveTo(Vector3.new(-25.95, 84, 3537.55))
        task.wait(0.15)
        local seat = Instance.new("Seat")
        seat.Name = "invischair"
        seat.Anchored = false; seat.CanCollide = false; seat.Transparency = 1
        seat.Position = Vector3.new(-25.95, 84, 3537.55); seat.Parent = workspace
        local weld = Instance.new("Weld")
        weld.Part0 = seat; weld.Part1 = character:FindFirstChild("Torso") or character:FindFirstChild("UpperTorso")
        weld.Parent = seat
        task.wait(); seat.CFrame = savedPosition
        setCharacterTransparency(character, 0)
    else
        local invisChair = workspace:FindFirstChild("invischair")
        if invisChair then invisChair:Destroy() end
        if character then setCharacterTransparency(character, 0) end
    end
end

local rayParams = RaycastParams.new()
rayParams.FilterType = Enum.RaycastFilterType.Exclude

local function IsVisible(targetPart)
    if not getgenv().WallCheck then return true end
    local origin = Camera.CFrame.Position
    local direction = (targetPart.Position - origin)
    local ignoreList = {}
    for _, p in ipairs(Players:GetPlayers()) do if p.Character then table.insert(ignoreList, p.Character) end end
    rayParams.FilterDescendantsInstances = ignoreList
    local result = workspace:Raycast(origin, direction, rayParams)
    return result == nil
end

local function IsEnemy(p) return not (getgenv().TeamCheck and Allies[p.UserId]) end

local function AutoDetectAllies()
    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end
    local myPos = char.HumanoidRootPart.Position
    local potential = {}
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            local d = (p.Character.HumanoidRootPart.Position - myPos).Magnitude
            if d <= getgenv().ProximityRange then table.insert(potential, {p = p, d = d}) end
        end
    end
    table.sort(potential, function(a,b) return a.d < b.d end)
    local newA = {}
    for i=1, math.min(#potential, getgenv().MaxAllies) do newA[potential[i].p.UserId] = true end
    Allies = newA
end

LocalPlayer.CharacterAdded:Connect(function() 
    task.wait(1.5)
    AutoDetectAllies()
    if getgenv().Invisibility_Enabled then toggleInvisibility() end
end)

workspace.ChildAdded:Connect(function(obj)
    if getgenv().Quantum and obj:IsA("BasePart") then
        local t, d = nil, math.huge
        for _, v in pairs(Players:GetPlayers()) do
            if v ~= LocalPlayer and v.Character and v.Character:FindFirstChild("Head") and v.Character:FindFirstChild("Humanoid") and v.Character.Humanoid.Health > 0 then
                if IsEnemy(v) then
                    local dist = (LocalPlayer.Character.HumanoidRootPart.Position - v.Character.HumanoidRootPart.Position).Magnitude
                    if dist < d then d = dist; t = v end
                end
            end
        end
        if t then obj.CFrame = t.Character.Head.CFrame end
    end
end)

local oldIndex;
oldIndex = hookmetamethod(game, "__index", function(self, key)
    if not checkcaller() and getgenv().Magnet_Enabled and self == Mouse and (key == "Target" or key == "Hit") then
        local t, d = nil, math.huge
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LocalPlayer and p.Character and IsEnemy(p) and p.Character:FindFirstChild("HumanoidRootPart") and p.Character:FindFirstChild("Humanoid") and p.Character.Humanoid.Health > 0 then
                local root = p.Character.HumanoidRootPart
                if IsVisible(root) then 
                    local dist = (root.Position - LocalPlayer.Character.HumanoidRootPart.Position).Magnitude
                    if dist < d then d = dist; t = root end
                end
            end
        end
        if t then return (key == "Hit" and t.CFrame or t) end
    end
    return oldIndex(self, key)
end)

return {
    setCharacterTransparency = setCharacterTransparency,
    toggleInvisibility = toggleInvisibility,
    IsVisible = IsVisible,
    IsEnemy = IsEnemy,
    AutoDetectAllies = AutoDetectAllies,
    Allies = Allies
}

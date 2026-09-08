--========================================================
-- HAZE SEAS HUB - TEST BUILD
-- Roblox Studio / experiência própria ou ambiente autorizado
--========================================================

local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local PathfindingService = game:GetService("PathfindingService")

local Player = Players.LocalPlayer

--========================================================
-- CONFIG
--========================================================

local Config = {
    AutoBoss = false,
    AutoAttack = true,
    AttackDelay = 0.25,
    FollowDistance = 8,
    RepathDelay = 1,

    BossPriority = {
        "Saturn",
        "Zenith",
        "Enma",
        "Dark Blade",
        "Ghost Ship",
        "Sea Beast",
    },

    -- Na sua experiência, coloque os NPCs dentro de:
    -- workspace.Bosses
    BossFolderName = "Bosses",
}

local State = {
    CurrentBoss = nil,
    CurrentTool = nil,
    Running = false,
    Path = nil,
}


--========================================================
-- DIMENSION / MOB FARM CONFIG
-- Para experiência própria / ambiente autorizado.
--
-- Estrutura esperada:
-- Workspace
-- └── Dimensions
--     ├── Sea1
--     │   ├── Cities      (Parts)
--     │   └── Mobs        (NPC Models)
--     ├── Sea2
--     │   ├── Cities
--     │   └── Mobs
--     └── Sea3
--         ├── Cities
--         └── Mobs
--========================================================

local DimensionConfig = {
    FolderName = "Dimensions",

    MobFarm = {
        Enabled = false,
        AttackDelay = 0.25,
        FollowDistance = 8,
        RepathDelay = 1,

        -- Dimensões serão descobertas automaticamente.
        -- Se uma dimensão não tiver "Mobs", ela simplesmente não aparece.
        CurrentDimension = nil,
        CurrentMob = nil,
    },
}

local function GetDimensionsFolder()
    return workspace:FindFirstChild(DimensionConfig.FolderName)
end

local function GetDimensions()
    local Result = {}
    local Folder = GetDimensionsFolder()

    if not Folder then
        return Result
    end

    for _, Dimension in ipairs(Folder:GetChildren()) do
        if Dimension:IsA("Folder") or Dimension:IsA("Model") then
            table.insert(Result, Dimension)
        end
    end

    table.sort(Result, function(a, b)
        return a.Name < b.Name
    end)

    return Result
end

local function GetMobsFolder(Dimension)
    if not Dimension then
        return nil
    end

    return Dimension:FindFirstChild("Mobs")
end

local function IsMobAlive(Mob)
    if not Mob or not Mob.Parent then
        return false
    end

    local Humanoid = Mob:FindFirstChildOfClass("Humanoid")

    if Humanoid then
        return Humanoid.Health > 0
    end

    local Health = Mob:GetAttribute("Health")

    if typeof(Health) == "number" then
        return Health > 0
    end

    return true
end

local function GetMobRoot(Mob)
    if not Mob then
        return nil
    end

    if Mob:IsA("Model") then
        return Mob:FindFirstChild("HumanoidRootPart")
            or Mob.PrimaryPart
    end

    if Mob:IsA("BasePart") then
        return Mob
    end

    return nil
end

local function FindNearestMob(Dimension)
    local Folder = GetMobsFolder(Dimension)
    local Root = GetRoot()

    if not Folder or not Root then
        return nil
    end

    local Best
    local BestDistance = math.huge

    for _, Mob in ipairs(Folder:GetChildren()) do
        if IsMobAlive(Mob) then
            local MobRoot = GetMobRoot(Mob)

            if MobRoot then
                local Distance = (Root.Position - MobRoot.Position).Magnitude

                if Distance < BestDistance then
                    BestDistance = Distance
                    Best = Mob
                end
            end
        end
    end

    return Best
end

local function MoveToMob(Mob)
    local Root = GetRoot()
    local MobRoot = GetMobRoot(Mob)

    if not Root or not MobRoot then
        return
    end

    local Distance = (Root.Position - MobRoot.Position).Magnitude

    if Distance <= DimensionConfig.MobFarm.FollowDistance then
        return
    end

    local Path = PathfindingService:CreatePath({
        AgentRadius = 2,
        AgentHeight = 5,
        AgentCanJump = true,
        AgentCanClimb = true,
        WaypointSpacing = 4,
    })

    local Success = pcall(function()
        Path:ComputeAsync(Root.Position, MobRoot.Position)
    end)

    if not Success or Path.Status ~= Enum.PathStatus.Success then
        return
    end

    for _, Waypoint in ipairs(Path:GetWaypoints()) do
        if not DimensionConfig.MobFarm.Enabled or not IsMobAlive(Mob) then
            break
        end

        local Humanoid = GetHumanoid()

        if not Humanoid then
            break
        end

        Humanoid:MoveTo(Waypoint.Position)

        if Waypoint.Action == Enum.PathWaypointAction.Jump then
            Humanoid.Jump = true
        end

        if not Humanoid.MoveToFinished:Wait() then
            break
        end
    end
end

local function GetMobDimensions()
    local Result = {}

    for _, Dimension in ipairs(GetDimensions()) do
        local Mobs = GetMobsFolder(Dimension)

        if Mobs and #Mobs:GetChildren() > 0 then
            table.insert(Result, Dimension)
        end
    end

    return Result
end

local function SelectMobDimension()
    if DimensionConfig.MobFarm.CurrentDimension then
        local Mobs = GetMobsFolder(DimensionConfig.MobFarm.CurrentDimension)

        if Mobs then
            return DimensionConfig.MobFarm.CurrentDimension
        end
    end

    local Dimensions = GetMobDimensions()

    return Dimensions[1]
end

local function AutoMobFarmLoop()
    if State.MobFarmRunning then
        return
    end

    State.MobFarmRunning = true

    while DimensionConfig.MobFarm.Enabled do
        local Dimension = SelectMobDimension()

        if not Dimension then
            task.wait(1)
            continue
        end

        DimensionConfig.MobFarm.CurrentDimension = Dimension

        local Mob = DimensionConfig.MobFarm.CurrentMob

        if not Mob or not IsMobAlive(Mob) then
            Mob = FindNearestMob(Dimension)
            DimensionConfig.MobFarm.CurrentMob = Mob
        end

        if Mob then
            EquipAttackTool()
            MoveToMob(Mob)

            while DimensionConfig.MobFarm.Enabled and IsMobAlive(Mob) do
                local Root = GetRoot()
                local MobRoot = GetMobRoot(Mob)

                if not Root or not MobRoot then
                    break
                end

                local Distance = (Root.Position - MobRoot.Position).Magnitude

                if Distance > DimensionConfig.MobFarm.FollowDistance then
                    MoveToMob(Mob)
                else
                    Attack()
                end

                task.wait(DimensionConfig.MobFarm.AttackDelay)
            end

            DimensionConfig.MobFarm.CurrentMob = nil
        else
            task.wait(1)
        end
    end

    State.MobFarmRunning = false
end

local MobFarm = {}

function MobFarm:Start()
    DimensionConfig.MobFarm.Enabled = true
    task.spawn(AutoMobFarmLoop)
end

function MobFarm:Stop()
    DimensionConfig.MobFarm.Enabled = false
    DimensionConfig.MobFarm.CurrentMob = nil
end

function MobFarm:SetDimension(Name)
    for _, Dimension in ipairs(GetDimensions()) do
        if string.lower(Dimension.Name) == string.lower(Name) then
            DimensionConfig.MobFarm.CurrentDimension = Dimension
            DimensionConfig.MobFarm.CurrentMob = nil
            return true
        end
    end

    return false
end


--========================================================
-- CHARACTER
--========================================================

local function GetCharacter()
    return Player.Character or Player.CharacterAdded:Wait()
end

local function GetHumanoid()
    return GetCharacter():FindFirstChildOfClass("Humanoid")
end

local function GetRoot()
    return GetCharacter():FindFirstChild("HumanoidRootPart")
end

--========================================================
-- WEAPONS
--========================================================

local function GetWeapons()
    local Weapons = {}
    local Backpack = Player:WaitForChild("Backpack")

    for _, Item in ipairs(Backpack:GetChildren()) do
        if Item:IsA("Tool") then
            table.insert(Weapons, Item)
        end
    end

    for _, Item in ipairs(GetCharacter():GetChildren()) do
        if Item:IsA("Tool") then
            table.insert(Weapons, Item)
        end
    end

    return Weapons
end

local function GetEquippedTool()
    for _, Item in ipairs(GetCharacter():GetChildren()) do
        if Item:IsA("Tool") then
            return Item
        end
    end
end

local function FindAttackTool()
    local Backpack = Player:WaitForChild("Backpack")

    -- Prioridade para espadas/blades
    for _, Tool in ipairs(Backpack:GetChildren()) do
        if Tool:IsA("Tool") then
            local Name = string.lower(Tool.Name)

            if string.find(Name, "sword", 1, true)
                or string.find(Name, "blade", 1, true)
                or string.find(Name, "katana", 1, true)
            then
                return Tool
            end
        end
    end

    -- Qualquer Tool como fallback
    for _, Tool in ipairs(Backpack:GetChildren()) do
        if Tool:IsA("Tool") then
            return Tool
        end
    end
end

local function EquipAttackTool()
    local Humanoid = GetHumanoid()
    if not Humanoid then
        return
    end

    local Equipped = GetEquippedTool()
    if Equipped then
        State.CurrentTool = Equipped
        return Equipped
    end

    local Tool = FindAttackTool()

    if Tool then
        Humanoid:EquipTool(Tool)
        task.wait(0.1)
        State.CurrentTool = GetEquippedTool()
        return State.CurrentTool
    end
end

local function Attack()
    if not Config.AutoAttack then
        return
    end

    local Tool = GetEquippedTool() or EquipAttackTool()

    if Tool and Tool.Enabled ~= false then
        Tool:Activate()
    end
end

--========================================================
-- BOSSES
--========================================================

local function GetBossFolder()
    return workspace:FindFirstChild(Config.BossFolderName)
end

local function GetBossHealth(Boss)
    local Humanoid = Boss:FindFirstChildOfClass("Humanoid")

    if Humanoid then
        return Humanoid.Health
    end

    local Health = Boss:GetAttribute("Health")
    if typeof(Health) == "number" then
        return Health
    end

    return nil
end

local function IsBossAlive(Boss)
    if not Boss or not Boss.Parent then
        return false
    end

    local Health = GetBossHealth(Boss)

    if Health ~= nil then
        return Health > 0
    end

    return true
end

local function FindBoss(Name)
    local Folder = GetBossFolder()

    if not Folder then
        return nil
    end

    for _, Boss in ipairs(Folder:GetChildren()) do
        if string.lower(Boss.Name) == string.lower(Name)
            and IsBossAlive(Boss)
        then
            return Boss
        end
    end

    return nil
end

local function FindPriorityBoss()
    for _, BossName in ipairs(Config.BossPriority) do
        local Boss = FindBoss(BossName)

        if Boss then
            return Boss
        end
    end

    return nil
end

local function GetBossRoot(Boss)
    if not Boss then
        return nil
    end

    if Boss:IsA("Model") then
        return Boss:FindFirstChild("HumanoidRootPart")
            or Boss.PrimaryPart
    end

    if Boss:IsA("BasePart") then
        return Boss
    end
end

--========================================================
-- MOVEMENT
--========================================================

local function MoveToBoss(Boss)
    local Root = GetRoot()
    local BossRoot = GetBossRoot(Boss)

    if not Root or not BossRoot then
        return
    end

    local Distance = (Root.Position - BossRoot.Position).Magnitude

    if Distance <= Config.FollowDistance then
        return
    end

    local Path = PathfindingService:CreatePath({
        AgentRadius = 2,
        AgentHeight = 5,
        AgentCanJump = true,
        AgentCanClimb = true,
        WaypointSpacing = 4,
    })

    State.Path = Path

    local Success = pcall(function()
        Path:ComputeAsync(Root.Position, BossRoot.Position)
    end)

    if not Success or Path.Status ~= Enum.PathStatus.Success then
        return
    end

    for _, Waypoint in ipairs(Path:GetWaypoints()) do
        if not Config.AutoBoss then
            break
        end

        if not IsBossAlive(Boss) then
            break
        end

        local Humanoid = GetHumanoid()
        if not Humanoid then
            break
        end

        Humanoid:MoveTo(Waypoint.Position)

        if Waypoint.Action == Enum.PathWaypointAction.Jump then
            Humanoid.Jump = true
        end

        local Reached = Humanoid.MoveToFinished:Wait()

        if not Reached then
            break
        end
    end
end

--========================================================
-- AUTO BOSS
--========================================================

local function AutoBossLoop()
    if State.Running then
        return
    end

    State.Running = true

    while Config.AutoBoss do
        local Boss = State.CurrentBoss

        if not Boss or not IsBossAlive(Boss) then
            Boss = FindPriorityBoss()
            State.CurrentBoss = Boss
        end

        if Boss then
            EquipAttackTool()
            MoveToBoss(Boss)

            while Config.AutoBoss and IsBossAlive(Boss) do
                local Root = GetRoot()
                local BossRoot = GetBossRoot(Boss)

                if not Root or not BossRoot then
                    break
                end

                local Distance =
                    (Root.Position - BossRoot.Position).Magnitude

                if Distance > Config.FollowDistance then
                    MoveToBoss(Boss)
                else
                    Attack()
                end

                task.wait(Config.AttackDelay)
            end

            State.CurrentBoss = nil
        else
            task.wait(1)
        end
    end

    State.Running = false
end


--========================================================
-- CITY TELEPORT / SERVER REJOIN
-- Para experiência própria / ambiente autorizado
--========================================================

local TeleportService = game:GetService("TeleportService")

-- Configure aqui os pontos de teleporte da sua experiência.
-- Crie uma pasta workspace.CitySpawns e coloque Parts dentro dela.
-- Ex.: workspace.CitySpawns["Starter City"], ["Marine City"], etc.
local CITY_FOLDER_NAME = "CitySpawns"

local function GetCityFolder()
    return workspace:FindFirstChild(CITY_FOLDER_NAME)
end

local function TeleportToCity(CityName)
    local Character = GetCharacter()
    local Root = Character and Character:FindFirstChild("HumanoidRootPart")
    local CityFolder = GetCityFolder()

    if not Root or not CityFolder then
        return false
    end

    local Spawn = CityFolder:FindFirstChild(CityName)

    if not Spawn or not Spawn:IsA("BasePart") then
        warn("[Haze Seas Hub] Cidade não encontrada: " .. tostring(CityName))
        return false
    end

    Root.CFrame = Spawn.CFrame + Vector3.new(0, 4, 0)
    return true
end

local function RejoinServer()
    local PlaceId = game.PlaceId

    local Success, ErrorMessage = pcall(function()
        TeleportService:Teleport(PlaceId, Player)
    end)

    if not Success then
        warn("[Haze Seas Hub] Falha ao reentrar: " .. tostring(ErrorMessage))
    end
end

--========================================================
-- CITY UI
--========================================================

local CityTitle = Instance.new("TextLabel")
CityTitle.Size = UDim2.new(1, -20, 0, 25)
CityTitle.Position = UDim2.fromOffset(10, 80)
CityTitle.BackgroundTransparency = 1
CityTitle.Text = "Cities"
CityTitle.TextColor3 = Color3.new(1, 1, 1)
CityTitle.TextSize = 14
CityTitle.Font = Enum.Font.GothamBold
CityTitle.TextXAlignment = Enum.TextXAlignment.Left
CityTitle.Parent = Main

local CityList = Instance.new("ScrollingFrame")
CityList.Size = UDim2.new(1, -20, 0, 100)
CityList.Position = UDim2.fromOffset(10, 105)
CityList.BackgroundColor3 = Color3.fromRGB(25, 25, 31)
CityList.BorderSizePixel = 0
CityList.ScrollBarThickness = 5
CityList.Parent = Main

Instance.new("UICorner", CityList).CornerRadius = UDim.new(0, 7)

local CityLayout = Instance.new("UIListLayout")
CityLayout.Padding = UDim.new(0, 5)
CityLayout.Parent = CityList

local function RefreshCities()
    for _, Child in ipairs(CityList:GetChildren()) do
        if Child:IsA("TextButton") then
            Child:Destroy()
        end
    end

    local CityFolder = GetCityFolder()

    if not CityFolder then
        return
    end

    for _, Spawn in ipairs(CityFolder:GetChildren()) do
        if Spawn:IsA("BasePart") then
            local Button = Instance.new("TextButton")
            Button.Size = UDim2.new(1, -10, 0, 32)
            Button.BackgroundColor3 = Color3.fromRGB(35, 35, 43)
            Button.BorderSizePixel = 0
            Button.Text = "  " .. Spawn.Name
            Button.TextColor3 = Color3.new(1, 1, 1)
            Button.TextSize = 13
            Button.Font = Enum.Font.Gotham
            Button.TextXAlignment = Enum.TextXAlignment.Left
            Button.Parent = CityList

            Instance.new("UICorner", Button).CornerRadius = UDim.new(0, 6)

            Button.MouseButton1Click:Connect(function()
                TeleportToCity(Spawn.Name)
            end)
        end
    end

    task.defer(function()
        CityList.CanvasSize = UDim2.fromOffset(
            0,
            CityLayout.AbsoluteContentSize.Y + 10
        )
    end)
end

local RejoinButton = Instance.new("TextButton")
RejoinButton.Size = UDim2.fromOffset(160, 36)
RejoinButton.Position = UDim2.new(1, -170, 0, 45)
RejoinButton.BackgroundColor3 = Color3.fromRGB(40, 40, 48)
RejoinButton.BorderSizePixel = 0
RejoinButton.Text = "Rejoin Server"
RejoinButton.TextColor3 = Color3.new(1, 1, 1)
RejoinButton.TextSize = 13
RejoinButton.Font = Enum.Font.GothamBold
RejoinButton.Parent = Main

Instance.new("UICorner", RejoinButton).CornerRadius = UDim.new(0, 7)


MobFarmToggle.MouseButton1Click:Connect(function()
    DimensionConfig.MobFarm.Enabled = not DimensionConfig.MobFarm.Enabled

    MobFarmToggle.Text =
        "Mob Farm: " .. (DimensionConfig.MobFarm.Enabled and "ON" or "OFF")

    if DimensionConfig.MobFarm.Enabled then
        task.spawn(AutoMobFarmLoop)
    else
        DimensionConfig.MobFarm.CurrentMob = nil
    end
end)

RejoinButton.MouseButton1Click:Connect(function()
    RejoinServer()
end)

RefreshCities()


--========================================================
-- DIMENSION-AWARE CITY TELEPORT
--========================================================

local function TeleportToDimensionCity(DimensionName, CityName)
    local Dimensions = GetDimensionsFolder()

    if not Dimensions then
        return false
    end

    local Dimension = Dimensions:FindFirstChild(DimensionName)
    if not Dimension then
        return false
    end

    local Cities = Dimension:FindFirstChild("Cities")
    if not Cities then
        return false
    end

    local Spawn = Cities:FindFirstChild(CityName)
    local Root = GetRoot()

    if not Spawn or not Spawn:IsA("BasePart") or not Root then
        return false
    end

    Root.CFrame = Spawn.CFrame + Vector3.new(0, 4, 0)
    return true
end

--========================================================
-- PROFESSIONAL GUI
--========================================================

local PlayerGui = Player:WaitForChild("PlayerGui")

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "HazeSeasHub"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = PlayerGui

-- Floating button used to reopen the minimized hub
local Reopen = Instance.new("TextButton")
Reopen.Name = "Reopen"
Reopen.Size = UDim2.fromOffset(54, 54)
Reopen.Position = UDim2.new(0, 20, 0.5, -27)
Reopen.BackgroundColor3 = Color3.fromRGB(24, 27, 36)
Reopen.BorderSizePixel = 0
Reopen.Text = "HS"
Reopen.TextColor3 = Color3.new(1, 1, 1)
Reopen.TextSize = 16
Reopen.Font = Enum.Font.GothamBold
Reopen.Visible = false
Reopen.Parent = ScreenGui

Instance.new("UICorner", Reopen).CornerRadius = UDim.new(1, 0)

local ReopenStroke = Instance.new("UIStroke")
ReopenStroke.Thickness = 1
ReopenStroke.Transparency = 0.2
ReopenStroke.Parent = Reopen

-- Main window
local Main = Instance.new("Frame")
Main.Name = "Main"
Main.Size = UDim2.fromOffset(680, 440)
Main.Position = UDim2.fromScale(0.5, 0.5)
Main.AnchorPoint = Vector2.new(0.5, 0.5)
Main.BackgroundColor3 = Color3.fromRGB(16, 18, 24)
Main.BorderSizePixel = 0
Main.Parent = ScreenGui

Instance.new("UICorner", Main).CornerRadius = UDim.new(0, 12)

local MainStroke = Instance.new("UIStroke")
MainStroke.Thickness = 1
MainStroke.Transparency = 0.25
MainStroke.Parent = Main

-- Top bar
local Topbar = Instance.new("Frame")
Topbar.Size = UDim2.new(1, 0, 0, 58)
Topbar.BackgroundColor3 = Color3.fromRGB(21, 24, 32)
Topbar.BorderSizePixel = 0
Topbar.Parent = Main

Instance.new("UICorner", Topbar).CornerRadius = UDim.new(0, 12)

local Accent = Instance.new("Frame")
Accent.Size = UDim2.fromOffset(4, 38)
Accent.Position = UDim2.fromOffset(8, 10)
Accent.BackgroundColor3 = Color3.fromRGB(90, 145, 255)
Accent.BorderSizePixel = 0
Accent.Parent = Topbar

Instance.new("UICorner", Accent).CornerRadius = UDim.new(1, 0)

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -170, 0, 24)
Title.Position = UDim2.fromOffset(24, 8)
Title.BackgroundTransparency = 1
Title.Text = "HAZE SEAS"
Title.TextColor3 = Color3.fromRGB(245, 245, 250)
Title.TextSize = 18
Title.Font = Enum.Font.GothamBold
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = Topbar

local Subtitle = Instance.new("TextLabel")
Subtitle.Size = UDim2.new(1, -170, 0, 18)
Subtitle.Position = UDim2.fromOffset(24, 31)
Subtitle.BackgroundTransparency = 1
Subtitle.Text = "Hub"
Subtitle.TextColor3 = Color3.fromRGB(145, 150, 165)
Subtitle.TextSize = 11
Subtitle.Font = Enum.Font.Gotham
Subtitle.TextXAlignment = Enum.TextXAlignment.Left
Subtitle.Parent = Topbar

local Minimize = Instance.new("TextButton")
Minimize.Size = UDim2.fromOffset(38, 32)
Minimize.Position = UDim2.new(1, -88, 0, 13)
Minimize.BackgroundColor3 = Color3.fromRGB(38, 41, 51)
Minimize.BorderSizePixel = 0
Minimize.Text = "—"
Minimize.TextColor3 = Color3.new(1, 1, 1)
Minimize.TextSize = 18
Minimize.Font = Enum.Font.GothamBold
Minimize.Parent = Topbar

Instance.new("UICorner", Minimize).CornerRadius = UDim.new(0, 7)

local Close = Instance.new("TextButton")
Close.Size = UDim2.fromOffset(38, 32)
Close.Position = UDim2.new(1, -44, 0, 13)
Close.BackgroundColor3 = Color3.fromRGB(55, 38, 43)
Close.BorderSizePixel = 0
Close.Text = "×"
Close.TextColor3 = Color3.fromRGB(255, 220, 225)
Close.TextSize = 18
Close.Font = Enum.Font.GothamBold
Close.Parent = Topbar

Instance.new("UICorner", Close).CornerRadius = UDim.new(0, 7)

-- Sidebar
local Sidebar = Instance.new("Frame")
Sidebar.Size = UDim2.new(0, 155, 1, -58)
Sidebar.Position = UDim2.fromOffset(0, 58)
Sidebar.BackgroundColor3 = Color3.fromRGB(19, 21, 28)
Sidebar.BorderSizePixel = 0
Sidebar.Parent = Main

local SidebarLayout = Instance.new("UIListLayout")
SidebarLayout.Padding = UDim.new(0, 7)
SidebarLayout.Parent = Sidebar

local SidebarPadding = Instance.new("UIPadding")
SidebarPadding.PaddingTop = UDim.new(0, 15)
SidebarPadding.PaddingLeft = UDim.new(0, 10)
SidebarPadding.PaddingRight = UDim.new(0, 10)
SidebarPadding.Parent = Sidebar

local function SidebarButton(Text)
    local Button = Instance.new("TextButton")
    Button.Size = UDim2.new(1, 0, 0, 40)
    Button.BackgroundColor3 = Color3.fromRGB(29, 32, 41)
    Button.BorderSizePixel = 0
    Button.Text = Text
    Button.TextColor3 = Color3.fromRGB(220, 222, 230)
    Button.TextSize = 13
    Button.Font = Enum.Font.GothamMedium
    Button.TextXAlignment = Enum.TextXAlignment.Left
    Button.Parent = Sidebar
    Instance.new("UICorner", Button).CornerRadius = UDim.new(0, 7)
    return Button
end

local EventsTab = SidebarButton("  Events")
local MobFarmTab = SidebarButton("  Mob Farm")
local WeaponsTab = SidebarButton("  Weapons")
local CitiesTab = SidebarButton("  Cities")
local SettingsTab = SidebarButton("  Settings")

-- Content
local Content = Instance.new("Frame")
Content.Size = UDim2.new(1, -155, 1, -58)
Content.Position = UDim2.fromOffset(155, 58)
Content.BackgroundColor3 = Color3.fromRGB(16, 18, 24)
Content.BorderSizePixel = 0
Content.Parent = Main

local PageTitle = Instance.new("TextLabel")
PageTitle.Size = UDim2.new(1, -30, 0, 30)
PageTitle.Position = UDim2.fromOffset(15, 12)
PageTitle.BackgroundTransparency = 1
PageTitle.Text = "Events"
PageTitle.TextColor3 = Color3.fromRGB(245, 245, 250)
PageTitle.TextSize = 17
PageTitle.Font = Enum.Font.GothamBold
PageTitle.TextXAlignment = Enum.TextXAlignment.Left
PageTitle.Parent = Content

local Status = Instance.new("TextLabel")
Status.Size = UDim2.new(1, -30, 0, 20)
Status.Position = UDim2.fromOffset(15, 40)
Status.BackgroundTransparency = 1
Status.Text = "Boss: nenhum"
Status.TextColor3 = Color3.fromRGB(145, 150, 165)
Status.TextSize = 12
Status.Font = Enum.Font.Gotham
Status.TextXAlignment = Enum.TextXAlignment.Left
Status.Parent = Content

local Controls = Instance.new("Frame")
Controls.Size = UDim2.new(1, -30, 0, 45)
Controls.Position = UDim2.fromOffset(15, 68)
Controls.BackgroundTransparency = 1
Controls.Parent = Content

local function ControlButton(Text, X, Width)
    local Button = Instance.new("TextButton")
    Button.Size = UDim2.fromOffset(Width, 38)
    Button.Position = UDim2.fromOffset(X, 0)
    Button.BackgroundColor3 = Color3.fromRGB(34, 37, 47)
    Button.BorderSizePixel = 0
    Button.Text = Text
    Button.TextColor3 = Color3.fromRGB(235, 235, 240)
    Button.TextSize = 12
    Button.Font = Enum.Font.GothamBold
    Button.Parent = Controls
    Instance.new("UICorner", Button).CornerRadius = UDim.new(0, 7)
    return Button
end

local AutoBossButton = ControlButton("Auto Boss: OFF", 0, 145)
local AutoAttackButton = ControlButton("Auto Attack: ON", 153, 145)
local RefreshButton = ControlButton("Refresh", 306, 110)

-- Pages
local function NewPage(Name)
    local Page = Instance.new("Frame")
    Page.Name = Name
    Page.Size = UDim2.new(1, -30, 1, -130)
    Page.Position = UDim2.fromOffset(15, 125)
    Page.BackgroundTransparency = 1
    Page.Visible = false
    Page.Parent = Content
    return Page
end

local EventsPage = NewPage("EventsPage")
local MobFarmPage = NewPage("MobFarmPage")
local WeaponsPage = NewPage("WeaponsPage")
local CitiesPage = NewPage("CitiesPage")
local SettingsPage = NewPage("SettingsPage")

EventsPage.Visible = true

local function MakeScroll(Page)
    local Scroll = Instance.new("ScrollingFrame")
    Scroll.Size = UDim2.fromScale(1, 1)
    Scroll.BackgroundColor3 = Color3.fromRGB(21, 23, 30)
    Scroll.BorderSizePixel = 0
    Scroll.ScrollBarThickness = 4
    Scroll.Parent = Page
    Instance.new("UICorner", Scroll).CornerRadius = UDim.new(0, 8)
    return Scroll
end

local BossList = MakeScroll(EventsPage)
local Layout = Instance.new("UIListLayout")
Layout.Padding = UDim.new(0, 6)
Layout.Parent = BossList
local BossPadding = Instance.new("UIPadding")
BossPadding.PaddingTop = UDim.new(0, 8)
BossPadding.PaddingLeft = UDim.new(0, 8)
BossPadding.PaddingRight = UDim.new(0, 8)
BossPadding.Parent = BossList


local MobFarmControls = Instance.new("Frame")
MobFarmControls.Size = UDim2.new(1, 0, 0, 45)
MobFarmControls.BackgroundTransparency = 1
MobFarmControls.Parent = MobFarmPage

local MobFarmToggle = Instance.new("TextButton")
MobFarmToggle.Size = UDim2.fromOffset(125, 34)
MobFarmToggle.BackgroundColor3 = Color3.fromRGB(34, 37, 47)
MobFarmToggle.BorderSizePixel = 0
MobFarmToggle.Text = "Farm  OFF"
MobFarmToggle.TextColor3 = Color3.new(1, 1, 1)
MobFarmToggle.TextSize = 12
MobFarmToggle.Font = Enum.Font.GothamBold
MobFarmToggle.Parent = MobFarmControls
Instance.new("UICorner", MobFarmToggle).CornerRadius = UDim.new(0, 7)

local MobDimensionList = Instance.new("ScrollingFrame")
MobDimensionList.Size = UDim2.new(1, 0, 1, -55)
MobDimensionList.Position = UDim2.fromOffset(0, 55)
MobDimensionList.BackgroundColor3 = Color3.fromRGB(21, 23, 30)
MobDimensionList.BorderSizePixel = 0
MobDimensionList.ScrollBarThickness = 4
MobDimensionList.Parent = MobFarmPage
Instance.new("UICorner", MobDimensionList).CornerRadius = UDim.new(0, 8)

local MobDimensionLayout = Instance.new("UIListLayout")
MobDimensionLayout.Padding = UDim.new(0, 6)
MobDimensionLayout.Parent = MobDimensionList

local function RefreshMobDimensions()
    for _, Child in ipairs(MobDimensionList:GetChildren()) do
        if Child:IsA("TextButton") then
            Child:Destroy()
        end
    end

    for _, Dimension in ipairs(GetMobDimensions()) do
        local Button = Instance.new("TextButton")
        Button.Size = UDim2.new(1, -8, 0, 40)
        Button.BackgroundColor3 = Color3.fromRGB(30, 33, 42)
        Button.BorderSizePixel = 0
        Button.Text = "  " .. Dimension.Name
        Button.TextColor3 = Color3.fromRGB(230, 232, 238)
        Button.TextSize = 13
        Button.Font = Enum.Font.Gotham
        Button.TextXAlignment = Enum.TextXAlignment.Left
        Button.Parent = MobDimensionList
        Instance.new("UICorner", Button).CornerRadius = UDim.new(0, 7)

        Button.MouseButton1Click:Connect(function()
            MobFarm:SetDimension(Dimension.Name)
        end)
    end

    task.defer(function()
        MobDimensionList.CanvasSize = UDim2.fromOffset(
            0,
            MobDimensionLayout.AbsoluteContentSize.Y + 15
        )
    end)
end

local WeaponTitle = Instance.new("TextLabel")
WeaponTitle.Size = UDim2.new(1, 0, 0, 28)
WeaponTitle.BackgroundTransparency = 1
WeaponTitle.Text = "Inventory / Weapons"
WeaponTitle.TextColor3 = Color3.fromRGB(240, 240, 245)
WeaponTitle.TextSize = 15
WeaponTitle.Font = Enum.Font.GothamBold
WeaponTitle.TextXAlignment = Enum.TextXAlignment.Left
WeaponTitle.Parent = WeaponsPage

local WeaponList = MakeScroll(WeaponsPage)
WeaponList.Position = UDim2.fromOffset(0, 35)
WeaponList.Size = UDim2.new(1, 0, 1, -35)
local WeaponLayout = Instance.new("UIListLayout")
WeaponLayout.Padding = UDim.new(0, 6)
WeaponLayout.Parent = WeaponList

local CityTitle = Instance.new("TextLabel")
CityTitle.Size = UDim2.new(1, 0, 0, 28)
CityTitle.BackgroundTransparency = 1
CityTitle.Text = "Cities / Teleport"
CityTitle.TextColor3 = Color3.fromRGB(240, 240, 245)
CityTitle.TextSize = 15
CityTitle.Font = Enum.Font.GothamBold
CityTitle.TextXAlignment = Enum.TextXAlignment.Left
CityTitle.Parent = CitiesPage

local CityList = MakeScroll(CitiesPage)
CityList.Position = UDim2.fromOffset(0, 35)
CityList.Size = UDim2.new(1, 0, 1, -35)
local CityLayout = Instance.new("UIListLayout")
CityLayout.Padding = UDim.new(0, 6)
CityLayout.Parent = CityList

local SettingsTitle = Instance.new("TextLabel")
SettingsTitle.Size = UDim2.new(1, 0, 0, 28)
SettingsTitle.BackgroundTransparency = 1
SettingsTitle.Text = "Settings"
SettingsTitle.TextColor3 = Color3.fromRGB(240, 240, 245)
SettingsTitle.TextSize = 15
SettingsTitle.Font = Enum.Font.GothamBold
SettingsTitle.TextXAlignment = Enum.TextXAlignment.Left
SettingsTitle.Parent = SettingsPage

local RejoinButton = Instance.new("TextButton")
RejoinButton.Size = UDim2.fromOffset(180, 40)
RejoinButton.Position = UDim2.fromOffset(0, 42)
RejoinButton.BackgroundColor3 = Color3.fromRGB(34, 37, 47)
RejoinButton.BorderSizePixel = 0
RejoinButton.Text = "Rejoin Server"
RejoinButton.TextColor3 = Color3.fromRGB(235, 235, 240)
RejoinButton.TextSize = 13
RejoinButton.Font = Enum.Font.GothamBold
RejoinButton.Parent = SettingsPage
Instance.new("UICorner", RejoinButton).CornerRadius = UDim.new(0, 7)

--========================================================
-- PAGE SWITCHING
--========================================================

local Pages = {
    Events = EventsPage,
    MobFarm = MobFarmPage,
    Weapons = WeaponsPage,
    Cities = CitiesPage,
    Settings = SettingsPage,
}

local function ShowPage(Name)
    for PageName, Page in pairs(Pages) do
        Page.Visible = (PageName == Name)
    end

    PageTitle.Text = Name
    Controls.Visible = (Name == "Events")
end

EventsTab.MouseButton1Click:Connect(function() ShowPage("Events") end)
MobFarmTab.MouseButton1Click:Connect(function() ShowPage("MobFarm") end)
WeaponsTab.MouseButton1Click:Connect(function() ShowPage("Weapons") end)
CitiesTab.MouseButton1Click:Connect(function() ShowPage("Cities") end)
SettingsTab.MouseButton1Click:Connect(function() ShowPage("Settings") end)

--========================================================
-- LIST REFRESHERS
--========================================================

local function RefreshWeaponList()
    for _, Child in ipairs(WeaponList:GetChildren()) do
        if Child:IsA("TextButton") then Child:Destroy() end
    end

    for _, Tool in ipairs(GetWeapons()) do
        local Button = Instance.new("TextButton")
        Button.Size = UDim2.new(1, -8, 0, 40)
        Button.BackgroundColor3 = Color3.fromRGB(30, 33, 42)
        Button.BorderSizePixel = 0
        Button.Text = "  " .. Tool.Name
        Button.TextColor3 = Color3.fromRGB(230, 232, 238)
        Button.TextSize = 13
        Button.Font = Enum.Font.Gotham
        Button.TextXAlignment = Enum.TextXAlignment.Left
        Button.Parent = WeaponList
        Instance.new("UICorner", Button).CornerRadius = UDim.new(0, 7)

        Button.MouseButton1Click:Connect(function()
            local Humanoid = GetHumanoid()
            if Humanoid then Humanoid:EquipTool(Tool) end
        end)
    end

    task.defer(function()
        WeaponList.CanvasSize = UDim2.fromOffset(0, WeaponLayout.AbsoluteContentSize.Y + 15)
    end)
end

local function RefreshCities()
    for _, Child in ipairs(CityList:GetChildren()) do
        if Child:IsA("TextButton") then Child:Destroy() end
    end

    local Folder = GetCityFolder()
    if not Folder then return end

    for _, Spawn in ipairs(Folder:GetChildren()) do
        if Spawn:IsA("BasePart") then
            local Button = Instance.new("TextButton")
            Button.Size = UDim2.new(1, -8, 0, 40)
            Button.BackgroundColor3 = Color3.fromRGB(30, 33, 42)
            Button.BorderSizePixel = 0
            Button.Text = "  " .. Spawn.Name
            Button.TextColor3 = Color3.fromRGB(230, 232, 238)
            Button.TextSize = 13
            Button.Font = Enum.Font.Gotham
            Button.TextXAlignment = Enum.TextXAlignment.Left
            Button.Parent = CityList
            Instance.new("UICorner", Button).CornerRadius = UDim.new(0, 7)

            Button.MouseButton1Click:Connect(function()
                TeleportToCity(Spawn.Name)
            end)
        end
    end

    task.defer(function()
        CityList.CanvasSize = UDim2.fromOffset(0, CityLayout.AbsoluteContentSize.Y + 15)
    end)
end

local function RefreshBossList()
    for _, Child in ipairs(BossList:GetChildren()) do
        if Child:IsA("TextButton") then Child:Destroy() end
    end

    for Index, BossName in ipairs(Config.BossPriority) do
        local Boss = FindBoss(BossName)

        local Button = Instance.new("TextButton")
        Button.LayoutOrder = Index
        Button.Size = UDim2.new(1, -8, 0, 40)
        Button.BackgroundColor3 = Color3.fromRGB(30, 33, 42)
        Button.BorderSizePixel = 0
        Button.Text = "  " .. BossName ..
            (Boss and "    [SPAWN]" or "    [WAITING]")
        Button.TextColor3 = Boss
            and Color3.fromRGB(170, 220, 175)
            or Color3.fromRGB(175, 178, 188)
        Button.TextSize = 13
        Button.Font = Enum.Font.GothamMedium
        Button.TextXAlignment = Enum.TextXAlignment.Left
        Button.Parent = BossList
        Instance.new("UICorner", Button).CornerRadius = UDim.new(0, 7)

        Button.MouseButton1Click:Connect(function()
            if Boss and IsBossAlive(Boss) then
                State.CurrentBoss = Boss
                Status.Text = "Boss: " .. Boss.Name
            end
        end)
    end

    task.defer(function()
        BossList.CanvasSize = UDim2.fromOffset(0, Layout.AbsoluteContentSize.Y + 20)
    end)
end

--========================================================
-- CONTROLS
--========================================================

AutoBossButton.MouseButton1Click:Connect(function()
    Config.AutoBoss = not Config.AutoBoss
    AutoBossButton.Text = "Auto Boss: " .. (Config.AutoBoss and "ON" or "OFF")

    if Config.AutoBoss then
        task.spawn(AutoBossLoop)
    else
        State.CurrentBoss = nil
    end
end)

AutoAttackButton.MouseButton1Click:Connect(function()
    Config.AutoAttack = not Config.AutoAttack
    AutoAttackButton.Text = "Auto Attack: " .. (Config.AutoAttack and "ON" or "OFF")
end)

RefreshButton.MouseButton1Click:Connect(function()
    EquipAttackTool()
    RefreshWeaponList()
    RefreshBossList()
    RefreshCities()
    RefreshMobDimensions()
end)


MobFarmToggle.MouseButton1Click:Connect(function()
    DimensionConfig.MobFarm.Enabled = not DimensionConfig.MobFarm.Enabled

    MobFarmToggle.Text =
        "Mob Farm: " .. (DimensionConfig.MobFarm.Enabled and "ON" or "OFF")

    if DimensionConfig.MobFarm.Enabled then
        task.spawn(AutoMobFarmLoop)
    else
        DimensionConfig.MobFarm.CurrentMob = nil
    end
end)

RejoinButton.MouseButton1Click:Connect(function()
    RejoinServer()
end)

--========================================================
-- MINIMIZE / REOPEN
--========================================================

local Minimized = false

local function SetMinimized(Value)
    Minimized = Value
    Main.Visible = not Value
    Reopen.Visible = Value
end

Minimize.MouseButton1Click:Connect(function()
    SetMinimized(true)
end)

Reopen.MouseButton1Click:Connect(function()
    SetMinimized(false)
end)

Close.MouseButton1Click:Connect(function()
    Config.AutoBoss = false
    State.CurrentBoss = nil
    ScreenGui:Destroy()
end)

--========================================================
-- DRAGGING
--========================================================

local UserInputService = game:GetService("UserInputService")
local Dragging = false
local DragStart
local StartPosition

Topbar.InputBegan:Connect(function(Input)
    if Input.UserInputType == Enum.UserInputType.MouseButton1
        or Input.UserInputType == Enum.UserInputType.Touch then

        Dragging = true
        DragStart = Input.Position
        StartPosition = Main.Position

        Input.Changed:Connect(function()
            if Input.UserInputState == Enum.UserInputState.End then
                Dragging = false
            end
        end)
    end
end)

UserInputService.InputChanged:Connect(function(Input)
    if not Dragging then return end

    if Input.UserInputType == Enum.UserInputType.MouseMovement
        or Input.UserInputType == Enum.UserInputType.Touch then

        local Delta = Input.Position - DragStart

        Main.Position = UDim2.new(
            StartPosition.X.Scale,
            StartPosition.X.Offset + Delta.X,
            StartPosition.Y.Scale,
            StartPosition.Y.Offset + Delta.Y
        )
    end
end)

--========================================================
-- LIVE STATUS
--========================================================

task.spawn(function()
    while ScreenGui.Parent do
        local Boss = State.CurrentBoss

        if Boss and IsBossAlive(Boss) then
            Status.Text = "Boss: " .. Boss.Name
        else
            Status.Text = "Boss: procurando..."
        end

        task.wait(0.5)
    end
end)

local Backpack = Player:WaitForChild("Backpack")

Backpack.ChildAdded:Connect(function(Item)
    if Item:IsA("Tool") then
        task.wait()
        RefreshWeaponList()
    end
end)

Backpack.ChildRemoved:Connect(function(Item)
    if Item:IsA("Tool") then
        task.wait()
        RefreshWeaponList()
    end
end)

Player.CharacterAdded:Connect(function()
    task.wait(1)
    RefreshWeaponList()
    RefreshBossList()
    RefreshMobDimensions()
end)

RefreshWeaponList()
RefreshBossList()
RefreshCities()
RefreshMobDimensions()
ShowPage("Events")

print("[Haze Seas Hub] Professional UI loaded.")

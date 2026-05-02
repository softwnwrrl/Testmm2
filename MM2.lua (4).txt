-- LocalScript: StarterPlayerScripts

if _G.__MurderHUD_Running then return end
_G.__MurderHUD_Running = true

local WALK_LEAD = 4.6
local WALK_LEAD_SLOW = 1.4
local WALK_LEAD_THROW = 0.4
local BULLET_DELAY    = 0.4
local VEL_SMOOTH_SIZE  = 4
local SPAM_JUMP_VEL    = 35
local FAKEBOMB_Y_OFFSET = 2.7
local lpLastActiveTime  = 0
local IDLE_KILLALL_DELAY = 30
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local UIS = game:GetService("UserInputService")
local GRAVITY         = workspace.Gravity

local lp = Players.LocalPlayer

-- ── State ─────────────────────────────────────────────────────────────────────
local roles             = {}
local stickyRoles       = {}
local visuals           = {}
local lpVisuals         = {}
local murderer          = nil
local isLpMurd          = false
local isLpSheriff = false
local gunDropHighlights = {}
local gunDropped      = false
local roundActive       = false
local murderGui = nil
local innocentGui = nil
local function isInLobby(char)
    if not char then return false end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return false end
    local lobbyParts = Workspace:FindFirstChild("RegularLobby")
        and Workspace.RegularLobby:FindFirstChild("MainLobby")
        and Workspace.RegularLobby.MainLobby:FindFirstChild("Parts")
    if not lobbyParts then return false end
    local ok, cf, size = pcall(function() return lobbyParts:GetBoundingBox() end)
    if not ok then return false end
    local localPos = cf:PointToObjectSpace(hrp.Position)
    local half = size / 2
    return math.abs(localPos.X) <= half.X
       and math.abs(localPos.Y) <= half.Y
       and math.abs(localPos.Z) <= half.Z
end

local ROLE_COLOR = {
    murder  = Color3.fromRGB(255, 0, 0),
    sheriff = Color3.fromRGB(0, 100, 255),
    hero    = Color3.fromRGB(255, 255, 0),
}
local LP_COLOR = {
    norole  = Color3.fromRGB(0, 255, 80),
    sheriff = Color3.fromRGB(0, 100, 255),
}

local rayParams      = RaycastParams.new()
rayParams.FilterType = Enum.RaycastFilterType.Exclude

local HIDE_POS2 = Vector3.new(0, -9999, 0)
local REAL_HRP_SIZE = Vector3.new(15, 5, 15)
local FAKE_HRP_SIZE = Vector3.new(2, 2, 1)

local fakeHRPs  = {}
local charParts = {}
local velSmooth = {}

-- ── Gun drop ESP ──────────────────────────────────────────────────────────────
local function attachGunDropHighlight(part)
    if gunDropHighlights[part] then return end
    local ok, err = pcall(function()
        local color = Color3.fromRGB(0, 255, 80)
        local bb = Instance.new("BillboardGui")
        bb.Name        = "GunDropTracker"
        bb.Adornee     = part
        bb.AlwaysOnTop = true
        bb.Size        = UDim2.new(0, 5, 0, 5)
        bb.StudsOffset = Vector3.new(0, 1, 0)
        bb.Parent      = lp.PlayerGui
        local frame = Instance.new("Frame", bb)
        frame.ZIndex               = 10
        frame.BackgroundTransparency = 0.3
        frame.BackgroundColor3     = color
        frame.Size                 = UDim2.new(1, 0, 1, 0)
        local txt = Instance.new("TextLabel", bb)
        txt.ZIndex                 = 10
        txt.Text                   = "Gun"
        txt.BackgroundTransparency = 1
        txt.Position               = UDim2.new(0, 0, 0, -35)
        txt.Size                   = UDim2.new(1, 0, 10, 0)
        txt.Font                   = Enum.Font.ArialBold
        txt.TextSize               = 12
        txt.TextStrokeTransparency = 0.5
        txt.TextColor3             = color
        gunDropHighlights[part]    = bb
        part.AncestryChanged:Connect(function(_, parent)
            if parent then return end
            if bb and bb.Parent then bb:Destroy() end
            gunDropHighlights[part] = nil
        end)
    end)
    if not ok then warn("[MurderHUD] GunDrop highlight: " .. tostring(err)) end
end

-- ── Walk / Jump ───────────────────────────────────────────────────────────────
local function setWalkSpeed(char)
    local hum = char:FindFirstChildOfClass("Humanoid")
    if hum then
        hum.WalkSpeed = 19
        hum:GetPropertyChangedSignal("WalkSpeed"):Connect(function()
            if hum.WalkSpeed ~= 19 then hum.WalkSpeed = 19 end
        end)
    else
        char.ChildAdded:Connect(function(child)
            if child:IsA("Humanoid") then
                child.WalkSpeed = 19
                child:GetPropertyChangedSignal("WalkSpeed"):Connect(function()
                    if child.WalkSpeed ~= 19 then child.WalkSpeed = 19 end
                end)
            end
        end)
    end
end

local function setJumpPower(char)
    local hum = char:FindFirstChildOfClass("Humanoid")
    if hum then
        hum.UseJumpPower = true
        hum.JumpPower = 56
        hum:GetPropertyChangedSignal("JumpPower"):Connect(function()
            if hum.JumpPower ~= 56 then hum.JumpPower = 56 end
        end)
    else
        char.ChildAdded:Connect(function(child)
            if child:IsA("Humanoid") then
                child.UseJumpPower = true
                child.JumpPower = 56
                child:GetPropertyChangedSignal("JumpPower"):Connect(function()
                    if child.JumpPower ~= 56 then child.JumpPower = 56 end
                end)
            end
        end)
    end
end

-- ── LP murderer ESP ───────────────────────────────────────────────────────────
local function removeLpVisual(p)
    local lv = lpVisuals[p]
    if lv and lv.bb and lv.bb.Parent then lv.bb:Destroy() end
    lpVisuals[p] = nil
end

local function clearAllLpVisuals()
    for p in pairs(lpVisuals) do removeLpVisual(p) end
end

local function attachLpVisual(p, char, color)
    removeLpVisual(p)
    local head = char:FindFirstChild("Head")
    if not head then return end
    local ok, err = pcall(function()
        local bb = Instance.new("BillboardGui")
        bb.Name          = "LpEspTracker"
        bb.Adornee       = head
        bb.AlwaysOnTop   = true
        bb.ExtentsOffset = Vector3.new(0, 1, 0)
        bb.Size          = UDim2.new(0, 5, 0, 5)
        bb.StudsOffset   = Vector3.new(0, 1, 0)
        bb.Parent        = lp.PlayerGui
        local frame = Instance.new("Frame", bb)
        frame.ZIndex               = 10
        frame.BackgroundTransparency = 0.3
        frame.BackgroundColor3     = color
        frame.Size                 = UDim2.new(1, 0, 1, 0)
        local txt = Instance.new("TextLabel", bb)
        txt.ZIndex                 = 10
        txt.Text                   = p.Name
        txt.BackgroundTransparency = 1
        txt.Position               = UDim2.new(0, 0, 0, -35)
        txt.Size                   = UDim2.new(1, 0, 10, 0)
        txt.Font                   = Enum.Font.ArialBold
        txt.TextSize               = 12
        txt.TextStrokeTransparency = 0.5
        txt.TextColor3             = color
        lpVisuals[p] = { bb = bb, color = color }
    end)
    if not ok then warn("[MurderHUD] LpVisual: " .. tostring(err)) end
end

-- ── Role visuals ──────────────────────────────────────────────────────────────
local function removeVisuals(p)
    local v = visuals[p]
    if not v then return end
    if v.bb and v.bb.Parent then v.bb:Destroy() end
    visuals[p] = nil
end

local function attachVisuals(p, char, role)
    removeVisuals(p)
    local head = char:FindFirstChild("Head")
    if not head then return end
    local ok, err = pcall(function()
        local color = ROLE_COLOR[role]
        local bb = Instance.new("BillboardGui")
        bb.Name          = "tracker"
        bb.Adornee       = head
        bb.AlwaysOnTop   = true
        bb.ExtentsOffset = Vector3.new(0, 1, 0)
        bb.Size          = UDim2.new(0, 5, 0, 5)
        bb.StudsOffset   = Vector3.new(0, 1, 0)
        bb.Parent        = lp.PlayerGui
        local frame = Instance.new("Frame", bb)
        frame.ZIndex               = 10
        frame.BackgroundTransparency = 0.3
        frame.BackgroundColor3     = color
        frame.Size                 = UDim2.new(1, 0, 1, 0)
        local txt = Instance.new("TextLabel", bb)
        txt.ZIndex                 = 10
        txt.Text                   = p.Name
        txt.BackgroundTransparency = 1
        txt.Position               = UDim2.new(0, 0, 0, -35)
        txt.Size                   = UDim2.new(1, 0, 10, 0)
        txt.Font                   = Enum.Font.ArialBold
        txt.TextSize               = 12
        txt.TextStrokeTransparency = 0.5
        txt.TextColor3             = color
        visuals[p] = { bb = bb }
    end)
    if not ok then warn("[MurderHUD] RoleVisual: " .. tostring(err)) end
end

-- ── Role detection ────────────────────────────────────────────────────────────
local function getRole(p)
    local char      = p.Character
    local bp        = p:FindFirstChild("Backpack")
    local wsModel   = Workspace:FindFirstChild(p.Name)
    if not char then return stickyRoles[p] end
    local hasKnife = char:FindFirstChild("Knife")
        or (bp       and bp:FindFirstChild("Knife"))
        or (wsModel  and wsModel:FindFirstChild("Knife"))
    if hasKnife then
        stickyRoles[p] = "murder"
        return "murder"
    end
    local hasGun = char:FindFirstChild("Gun")
        or (bp       and bp:FindFirstChild("Gun"))
        or (wsModel  and wsModel:FindFirstChild("Gun"))
    if hasGun then
        local role = gunDropped and "hero" or "sheriff"
        stickyRoles[p] = role
        return role
    end
    return stickyRoles[p]
end

-- ── LP visual for one player ──────────────────────────────────────────────────
local function updateLpVisualFor(p)
    if not isLpMurd then removeLpVisual(p) return end
    local pChar = p.Character
    if not pChar then removeLpVisual(p) return end
    local role = roles[p]
    if role == "murder" then removeLpVisual(p) return end
    local lpColor = role == "sheriff" and LP_COLOR.sheriff or LP_COLOR.norole
    local lv = lpVisuals[p]
    if not lv or lv.color ~= lpColor then
        attachLpVisual(p, pChar, lpColor)
    end
end

local function endRound()
    if not roundActive then return end
    roundActive = false
    if murderGui then murderGui.Enabled = false end
    if innocentGui then innocentGui.Enabled = false end
    gunDropped = false
    murderer = nil
    task.delay(9, function()
        for p in pairs(visuals) do removeVisuals(p) end
        for p in pairs(lpVisuals) do removeLpVisual(p) end
        roles = {}
        stickyRoles = {}
    end)
end

local function checkInnocentsDead()
    if not roundActive then return end
    for _, p in ipairs(Players:GetPlayers()) do
        if p == murderer then continue end
        local char = p.Character
        local hum  = char and char:FindFirstChildOfClass("Humanoid")
        if hum and hum.Health > 0 then return end
    end
    endRound()
end

local function startRound()
    endRound()
    gunDropped       = false
    roundActive      = true
end

-- ── Apply role state for a player ─────────────────────────────────────────────
local function applyRole(p)
    local role  = getRole(p)
    local pChar = p.Character
    local old   = roles[p]
    roles[p] = role
    if role == "murder" then
        if murderer ~= p then
            murderer = p
            startRound()
        end
    elseif old == "murder" and murderer == p then
        murderer = nil
    end
    if role and pChar then
        if isInLobby(pChar) then
            removeVisuals(p)
            removeLpVisual(p)
        else
            local v = visuals[p]
            if not v or not v.bb or not v.bb.Parent or old ~= role then
                attachVisuals(p, pChar, role)
            end
        end
    else
        removeVisuals(p)
    end
    if old ~= role then
        updateLpVisualFor(p)
    end
end

-- ── LP murderer state ─────────────────────────────────────────────────────────
local function refreshLpMurd()
    local char = lp.Character
    local bp   = lp:FindFirstChild("Backpack")
    local prev = isLpMurd
    local wsModel = Workspace:FindFirstChild(lp.Name)
    isLpMurd = (char    and char:FindFirstChild("Knife")    ~= nil)
            or (bp      and bp:FindFirstChild("Knife")      ~= nil)
            or (wsModel and wsModel:FindFirstChild("Knife") ~= nil)
    if prev == isLpMurd then return end
    if isLpMurd then
        for _, p in ipairs(Players:GetPlayers()) do
            lpLastActiveTime = tick()
            if p ~= lp then
                applyRole(p)
                updateLpVisualFor(p)
            end
        end
        if innocentGui then innocentGui.Enabled = false end
    else
        clearAllLpVisuals()
    end
    local lpInLobby = isInLobby(lp.Character)
    if murderGui then murderGui.Enabled = isLpMurd and not lpInLobby end
end

-- ── Watch container for tool events ──────────────────────────────────────────
local function watchContainer(p, container, forLp)
    if forLp then
        container.ChildAdded:Connect(function(child)
            if child.Name == "Knife" then refreshLpMurd() end
        end)
        container.ChildRemoved:Connect(function(child)
            if child.Name == "Knife" then refreshLpMurd() end
        end)
    else
        container.ChildAdded:Connect(function(child)
            if child.Name == "Knife" or child.Name == "Gun" then applyRole(p) end
        end)
        container.ChildRemoved:Connect(function(child)
            if child.Name == "Knife" or child.Name == "Gun" then applyRole(p) end
        end)
    end
end

-- ── Watch character for removal ───────────────────────────────────────────────
local function watchChar(p, char)
    local hum = char:FindFirstChildOfClass("Humanoid")
    if hum then
        hum.Died:Connect(function()
            removeLpVisual(p)
            removeVisuals(p)
            if murderer == p then
                murderer = nil
                gunDropped = false
                for _, pl in ipairs(Players:GetPlayers()) do
                    if pl ~= lp then
                        task.wait(6)
                        stickyRoles[pl] = nil
                        applyRole(pl)
                        updateLpVisualFor(pl)
                    end
                end
                endRound()     
            end
        end)
    end
    char.AncestryChanged:Connect(function(_, parent)
        if parent ~= nil then
            if isInLobby(char) then
                removeLpVisual(p)
                removeVisuals(p)
            end
            return
        end
        roles[p]       = nil
        stickyRoles[p] = nil
        if murderer == p then
            murderer = nil
            endRound()
        else
            checkInnocentsDead()
        end
    end)
end

-- ── FakeHRP helpers ───────────────────────────────────────────────────────────
local function rebuildCharParts(p)
    local char = p.Character
    if not char then charParts[p] = nil return end
    local list = {}
    for _, v in ipairs(char:GetDescendants()) do
        if v:IsA("BasePart") then list[#list + 1] = v end
    end
    charParts[p] = list
    char.DescendantAdded:Connect(function(v)
        if v:IsA("BasePart") then
            local l = charParts[p]
            if l then l[#l + 1] = v end
        end
    end)
end

local function expandRealHRP(p)
    local char = p.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if hrp then
        hrp.Size       = REAL_HRP_SIZE
        hrp.CanCollide = false
    end
end

local function ensureFakeHRP(p)
    if fakeHRPs[p] and fakeHRPs[p].Parent then return end
    local ok, err = pcall(function()
        local part = Instance.new("Part")
        part.Name         = "FakeHRP_" .. p.Name
        part.Anchored     = true
        part.CanCollide   = true
        part.CanQuery     = false
        part.CanTouch     = false
        part.Transparency = 1
        part.CastShadow   = false
        part.Size         = FAKE_HRP_SIZE
        part.CFrame       = CFrame.new(HIDE_POS2)
        part.Parent       = Workspace
        fakeHRPs[p]       = part
    end)
    if not ok then warn("[MurderHUD] FakeHRP create: " .. tostring(err)) end
end

-- ── Per-player setup ──────────────────────────────────────────────────────────
local function setupPlayer(p)
    ensureFakeHRP(p)
    -- Watch persistent Backpack once
    local bp = p:FindFirstChild("Backpack")
    if bp then
        watchContainer(p, bp, false)
    else
        p.ChildAdded:Connect(function(child)
            if child.Name == "Backpack" then watchContainer(p, child, false) end
        end)
    end
    -- Watch current character
    if p.Character then
        watchChar(p, p.Character)
        watchContainer(p, p.Character, false)
        expandRealHRP(p)
        rebuildCharParts(p)
        applyRole(p)
        updateLpVisualFor(p)
    end
    -- Watch future characters (new char = new round, clear sticky)
    p.CharacterAdded:Connect(function(char)
        stickyRoles[p] = nil
        watchChar(p, char)
        watchContainer(p, char, false)
        local bp2 = p:FindFirstChild("Backpack")
        if bp2 then watchContainer(p, bp2, false) end
        expandRealHRP(p)
        rebuildCharParts(p)
        applyRole(p)
        updateLpVisualFor(p)
        task.delay(1, function() if p.Character == char then applyRole(p) end end)
        task.delay(3, function() if p.Character == char then applyRole(p) end end)
    end)
end

-- ── LP setup ──────────────────────────────────────────────────────────────────
local function setupLp()
    local char = lp.Character
    if char then
        setWalkSpeed(char)
        setJumpPower(char)
        watchContainer(lp, char, true)
    end
    local bp = lp:FindFirstChild("Backpack")
    if bp then
        watchContainer(lp, bp, true)
    end
    refreshLpMurd()
end

local function refreshLpSheriff()
    local char = lp.Character
    local bp   = lp:FindFirstChild("Backpack")
    local prev = isLpSheriff
    isLpSheriff = (char and char:FindFirstChild("Gun") ~= nil)
               or (bp   and bp:FindFirstChild("Gun")   ~= nil)
    if prev == isLpSheriff then return end
    local lpInLobby = isInLobby(lp.Character)
    if innocentGui then innocentGui.Enabled = not lpInLobby and not isLpMurd and not isLpSheriff and innocentGui.Enabled end
    if murderGui and lpInLobby then murderGui.Enabled = false end
end

local function watchLpGun(container)
    container.ChildAdded:Connect(function(child)
        if child.Name == "Gun" then refreshLpSheriff() end
    end)
    container.ChildRemoved:Connect(function(child)
        if child.Name == "Gun" then refreshLpSheriff() end
    end)
end

do
    local char = lp.Character
    if char then watchLpGun(char) end
    local bp = lp:FindFirstChild("Backpack")
    if bp then watchLpGun(bp) end
    refreshLpSheriff()
end

lp.ChildAdded:Connect(function(child)
    if child.Name == "Backpack" then
        watchContainer(lp, child, true)
        watchLpGun(child)
    end
end)

lp.CharacterAdded:Connect(function(char)
    setWalkSpeed(char)
    setJumpPower(char)
    watchContainer(lp, char, true)
    refreshLpMurd()
end)

setupLp()

for _, p in ipairs(Players:GetPlayers()) do
    if p ~= lp then setupPlayer(p) end
end

Players.PlayerAdded:Connect(function(p)
    if p == lp then return end
    setupPlayer(p)
end)

Players.PlayerRemoving:Connect(function(p)
    roles[p]       = nil
    stickyRoles[p] = nil
    velSmooth[p] = nil
    removeLpVisual(p)
    removeVisuals(p)
    if murderer == p then
        murderer = nil
        endRound()
    else
        checkInnocentsDead()
    end
    local fake = fakeHRPs[p]
    if fake and fake.Parent then fake:Destroy() end
    fakeHRPs[p]  = nil
    charParts[p] = nil
end)

-- ── FakeHRP sync: Heartbeat (positional, must remain per-frame) ───────────────
RunService.Heartbeat:Connect(function()
    for p, fakePart in pairs(fakeHRPs) do
        local char = p.Character
        local hrp  = char and char:FindFirstChild("HumanoidRootPart")
        -- Velocity smoothing sample
        if hrp then
            if not velSmooth[p] then velSmooth[p] = {} end
            local buf = velSmooth[p]
            table.insert(buf, hrp.AssemblyLinearVelocity)
            if #buf > VEL_SMOOTH_SIZE then table.remove(buf, 1) end
        end
        if hrp then
            fakePart.CFrame = hrp.CFrame
            if hrp.Size ~= REAL_HRP_SIZE then
                hrp.Size       = REAL_HRP_SIZE
                hrp.CanCollide = false
            end
            local list = charParts[p]
            if list then
                for i = 1, #list do
                    local v = list[i]
                    if v and v.Parent and v.CanCollide then
                        v.CanCollide = false
                    end
                end
            end
        else
            fakePart.CFrame = CFrame.new(HIDE_POS2)
        end
    end
end)

-- ── Aim: gun (targets murderer) ───────────────────────────────────────────────
local function getSmoothedVel(p)
    local buf = velSmooth[p]
    if not buf or #buf == 0 then
        local char = p.Character
        local hrp  = char and char:FindFirstChild("HumanoidRootPart")
        return hrp and hrp.AssemblyLinearVelocity or Vector3.zero
    end
    local sum = Vector3.zero
    for _, v in ipairs(buf) do sum = sum + v end
    return sum / #buf
end

local function getAimPosition()
    if not murderer then return nil end
    local char = murderer.Character
    if not char then return nil end

    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return nil end

    local torso = char:FindFirstChild("UpperTorso") or char:FindFirstChild("Torso")
    local head  = char:FindFirstChild("Head")
    local hum   = char:FindFirstChildOfClass("Humanoid")

    local myChar = lp.Character
    local myHRP  = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if not myHRP then return nil end

    local rawVel  = hrp.AssemblyLinearVelocity
    local smoothV = getSmoothedVel(murderer)

    local vel = smoothV * 0.6 + rawVel * 0.4

    local pos   = hrp.Position
    local hVel  = Vector3.new(vel.X, 0, vel.Z)
    local speed = hVel.Magnitude

    local isAir      = hum and hum.FloorMaterial == Enum.Material.Air
    local isClimbing = hum and hum:GetState() == Enum.HumanoidStateType.Climbing
    local inAir      = isAir and not isClimbing

    local torsoOff = torso and (torso.Position - pos) or Vector3.new(0, 0.9, 0)
    local headOff  = head  and (head.Position  - pos) or Vector3.new(0, 2.5, 0)

    local dist  = (pos - myHRP.Position).Magnitude
    local dt    = BULLET_DELAY + math.clamp(dist / 400, 0, 0.1)

    -- ── Idle on ground: shoot current torso with visibility check ───────────
    if speed < 1.5 and not inAir then
        local target = torso and torso.Position or (pos + torsoOff)
        rayParams.FilterDescendantsInstances = { myChar, char }
        local dir = target - myHRP.Position
        local hit = Workspace:Raycast(myHRP.Position, dir, rayParams)
        if not hit or hit.Instance:IsDescendantOf(char) then return target end
        if head then
            local hDir = head.Position - myHRP.Position
            local hHit = Workspace:Raycast(myHRP.Position, hDir, rayParams)
            if not hHit or hHit.Instance:IsDescendantOf(char) then return head.Position end
        end
        return target
    end

    -- ── Horizontal prediction (constant velocity, blended) ──────────────────
    local LEAD_FAST   = 4.6   -- speed >= 15.87
    local LEAD_SLOW   = 1.4   -- speed >= 9
    local LEAD_THROW  = 0.4   -- speed > 0 but slow

    local hUnit  = hVel.Magnitude > 0 and hVel.Unit or Vector3.zero
    local lead
    if speed >= 15.87 then
        lead = LEAD_FAST
    elseif speed >= 9 then
        lead = LEAD_SLOW
    elseif speed > 0 then
        lead = LEAD_THROW
    else
        lead = 0
    end

    local hOffset = hUnit * lead
    local predX   = pos.X + hOffset.X
    local predZ   = pos.Z + hOffset.Z

    -- ── Vertical prediction ─────────────────────────────────────────────────
    local predY
    if inAir then
        local velY = vel.Y

        -- Spam-jump: player just left ground with high upward velocity.
        -- Within dt they may reach apex and start falling.
        -- Predict to the apex if apex occurs within dt, otherwise full physics.
        if velY >= SPAM_JUMP_VEL then
            local tApex = velY / GRAVITY          -- time to reach apex
            if tApex <= dt then
                -- Will reach apex and fall for remaining (dt - tApex)
                local apexY   = pos.Y + velY * tApex - 0.5 * GRAVITY * tApex * tApex
                local fallDt  = dt - tApex
                predY = apexY - 0.5 * GRAVITY * fallDt * fallDt
            else
                -- Still ascending at bullet arrival
                predY = pos.Y + velY * dt - 0.5 * GRAVITY * dt * dt
            end
        elseif velY < 0 then
            -- Falling or at apex: check if they'd land before dt
            -- Conservative: full physics, but clamp so we don't go underground
            predY = pos.Y + velY * dt - 0.5 * GRAVITY * dt * dt
            -- If prediction goes below current ground level, clamp to HRP Y
            -- (spam-jumper will re-jump so aim near floor level)
            if predY < pos.Y - 8 then
                predY = pos.Y - 4   -- aim at waist height as they land/re-jump
            end
        else
            -- Slow ascent or near apex
            predY = pos.Y + velY * dt - 0.5 * GRAVITY * dt * dt
        end
    else
        predY = pos.Y  -- grounded: no vertical offset needed
    end

    local predHRP = Vector3.new(predX, predY, predZ)

    -- Predicted body-part positions
    local candidates = {
        predHRP + torsoOff,   -- primary
        predHRP + headOff,    -- secondary
        predHRP,              -- HRP fallback
    }

    -- ── Visibility: pick first unobstructed predicted point ─────────────────
    rayParams.FilterDescendantsInstances = { myChar, char }
    for _, cPos in ipairs(candidates) do
        local dir = cPos - myHRP.Position
        local hit = Workspace:Raycast(myHRP.Position, dir, rayParams)
        if not hit or hit.Instance:IsDescendantOf(char) then
            return cPos
        end
    end

    return candidates[1]  -- all blocked: best guess
end

-- ── Remote getters ────────────────────────────────────────────────────────────
local function getShootRemote()
    local char    = lp.Character
    local bp      = lp:FindFirstChild("Backpack")
    local wsModel = Workspace:FindFirstChild(lp.Name)
    local gun = (char    and char:FindFirstChild("Gun"))
             or (bp      and bp:FindFirstChild("Gun"))
             or (wsModel and wsModel:FindFirstChild("Gun"))
    if not gun then return nil end
    local r = gun:FindFirstChild("Shoot")
    return (r and r:IsA("RemoteEvent")) and r or nil
end

-- ── FakeBomb ──────────────────────────────────────────────────────────────────
local function doFakeBomb()
    task.wait(0.2)

    local char = lp.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    local bomb = char:FindFirstChild("FakeBomb")
        or (Workspace:FindFirstChild(lp.Name) and Workspace[lp.Name]:FindFirstChild("FakeBomb"))
    if not bomb then warn("[MurderHUD] FakeBomb: not found on character") return end

    local remote = bomb:FindFirstChild("Remote")
    if not (remote and remote:IsA("RemoteEvent")) then
        warn("[MurderHUD] FakeBomb: Remote not found")
        return
    end

    local placeCF = CFrame.new(Vector3.new(
        hrp.Position.X,
        hrp.Position.Y - FAKEBOMB_Y_OFFSET,
        hrp.Position.Z
    ))

    local ok, err = pcall(function()
        remote:FireServer(placeCF, 50)
    end)
    if not ok then warn("[MurderHUD] FakeBomb FireServer: " .. tostring(err)) end
end

UIS.JumpRequest:Connect(function()
    local char = lp.Character
    if not char then return end
    if not (char:FindFirstChild("FakeBomb")
        or (Workspace:FindFirstChild(lp.Name) and Workspace[lp.Name]:FindFirstChild("FakeBomb"))) then return end
    local ok, err = pcall(doFakeBomb)
    if not ok then warn("[MurderHUD] FakeBomb JumpRequest: " .. tostring(err)) end
end)

-- ── Input ─────────────────────────────────────────────────────────────────────
local touchStartPos = nil
UIS.InputBegan:Connect(function(input, processed)
    if input.UserInputType == Enum.UserInputType.Touch
    or input.UserInputType == Enum.UserInputType.MouseButton1 then
        touchStartPos = input.Position
        if isLpMurd then lpLastActiveTime = tick() end
    end
end)

UIS.InputEnded:Connect(function(input, processed)
    if processed then return end
    local isFire = input.UserInputType == Enum.UserInputType.MouseButton1
               or (input.UserInputType == Enum.UserInputType.Touch
                   and not UIS:GetFocusedTextBox()
                   and input.Position.X > (workspace.CurrentCamera.ViewportSize.X * 0.35))
    if not isFire then return end
    if touchStartPos then
        local delta = (Vector2.new(input.Position.X, input.Position.Y) - Vector2.new(touchStartPos.X, touchStartPos.Y)).Magnitude
        if delta > 12 then touchStartPos = nil return end
    end
    touchStartPos = nil
    local myChar = lp.Character
    local myHRP  = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if not myHRP then return end
    if not murderer then return end
    local wsModel = Workspace:FindFirstChild(lp.Name)
    if not wsModel or not wsModel:FindFirstChild("Gun") then return end
    local aimPos = getAimPosition()
    if not aimPos then return end
    local remote = getShootRemote()
    local ok, err = pcall(function()
        remote:FireServer(CFrame.new(myHRP.Position, aimPos), CFrame.new(aimPos))
    end)
end)

local function doThrowKnife()
    local char = lp.Character
    if not char then return end
    local myHRP = char:FindFirstChild("HumanoidRootPart")
    if not myHRP then return end
    local knife = char:FindFirstChild("Knife")
    if not knife then
        local bp = lp:FindFirstChild("Backpack")
        if bp then
            local bpKnife = bp:FindFirstChild("Knife")
            if bpKnife then
                local hum = char:FindFirstChildOfClass("Humanoid")
                if not hum then warn("[MurderHUD] ThrowKnife: no Humanoid") return end
                local ok2, err2 = pcall(function() hum:EquipTool(bpKnife) end)
                if not ok2 then warn("[MurderHUD] ThrowKnife equip: " .. tostring(err2)) return end
                task.wait(0.1)
                knife = char:FindFirstChild("Knife")
            end
        end
    end
    if not knife then warn("[MurderHUD] ThrowKnife: no Knife found") return end
    local knifeEvents = knife:FindFirstChild("Events")
    local throwRemote = knifeEvents and knifeEvents:FindFirstChild("KnifeThrown")
    if not (throwRemote and throwRemote:IsA("RemoteEvent")) then
        warn("[MurderHUD] ThrowKnife: KnifeThrown remote not found") return
    end
    local candidates = {}
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= lp and p.Character then
            local hrp2 = p.Character:FindFirstChild("HumanoidRootPart")
            if hrp2 then
                local dist = (hrp2.Position - myHRP.Position).Magnitude
                table.insert(candidates, { player = p, hrp = hrp2, dist = dist })
            end
        end
    end
    table.sort(candidates, function(a, b) return a.dist < b.dist end)
    local nearest, nearestHRP = nil, nil
    rayParams.FilterDescendantsInstances = { char }
    for _, c in ipairs(candidates) do
        local dir    = c.hrp.Position - myHRP.Position
        local result = Workspace:Raycast(myHRP.Position, dir, rayParams)
        if not result or (result.Instance and result.Instance:IsDescendantOf(c.player.Character)) then
            nearest    = c.player
            nearestHRP = c.hrp
            break
        end
    end
    if not nearest and #candidates > 0 then
        nearest    = candidates[1].player
        nearestHRP = candidates[1].hrp
    end
    if not nearest then warn("[MurderHUD] ThrowKnife: no target found") return end
    local vel  = nearestHRP.AssemblyLinearVelocity
    local hVel = Vector3.new(vel.X, 0, vel.Z)
    local aimPos = nearestHRP.Position
    if hVel.Magnitude > 0 then
        aimPos = aimPos + hVel.Unit * WALK_LEAD_THROW
    end
    local ok, err = pcall(function()
        throwRemote:FireServer(CFrame.new(myHRP.Position, aimPos), CFrame.new(aimPos))
    end)
    if not ok then warn("[MurderHUD] ThrowKnife FireServer: " .. tostring(err)) end
end

-- ── killall ────────────────────────────────────────────────────
local function doKillAll()
    local char = lp.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return end
    local knife = char:FindFirstChild("Knife")
    if not knife then
        local bp = lp:FindFirstChild("Backpack")
        if bp then
            local bpKnife = bp:FindFirstChild("Knife")
            if bpKnife then
                local ok2, err2 = pcall(function() hum:EquipTool(bpKnife) end)
                if not ok2 then warn("[MurderHUD] KillAll equip: " .. tostring(err2)) return end
                task.wait(0.1)
                knife = char:FindFirstChild("Knife")
            end
        end
    end
    if not knife then warn("[MurderHUD] KillAll: no Knife found") return end
    local knifeEvents = knife:FindFirstChild("Events")
    local stab = knifeEvents and knifeEvents:FindFirstChild("KnifeStabbed")
    if not (stab and stab:IsA("RemoteEvent")) then
        pcall(function() lp.Character:FindFirstChild("Knife").Events.KnifeStabbed:FireServer() end)
        return
    end
    local myHRP = char:FindFirstChild("HumanoidRootPart")
    if not myHRP then return end
    local hrpList = {}
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= lp and p.Character then
            local hrp = p.Character:FindFirstChild("HumanoidRootPart")
            if hrp then
                local ok3, err3 = pcall(function()
                    hrp.Anchored = true
                    hrp.CFrame = myHRP.CFrame + myHRP.CFrame.LookVector * 1
                end)
                if ok3 then table.insert(hrpList, hrp)
                else warn("[MurderHUD] KillAll anchor: " .. tostring(err3)) end
            end
        end
    end
    local ok4, err4 = pcall(function() stab:FireServer() end)
    if not ok4 then warn("[MurderHUD] KillAll FireServer: " .. tostring(err4)) end
    task.delay(2, function()
        for _, hrp in ipairs(hrpList) do
            pcall(function()
                if hrp and hrp.Parent then hrp.Anchored = false end
            end)
        end
    end)
end

local function doGrabGun()
    local char = lp.Character
    local hrp  = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local savedCFrame = hrp.CFrame
    local gunDrop = nil
    for _, desc in ipairs(Workspace:GetDescendants()) do
        if desc.Name == "GunDrop" and desc:IsA("BasePart") then
            gunDrop = desc
            break
        end
    end
    if not gunDrop then warn("[MurderHUD] GrabGun: no GunDrop found") return end
    local ok, err = pcall(function()
        hrp.CFrame = CFrame.new(gunDrop.Position)
        firetouchinterest(gunDrop, hrp, 0)
        hrp.CFrame = savedCFrame
    end)
    if not ok then warn("[MurderHUD] GrabGun: " .. tostring(err)) end
end

-- ── Murder GUI ────────────────────────────────────────────────────────────────
do
    local gui = Instance.new("ScreenGui")
    gui.Name        = "MurderHUD_Gui"
    gui.ResetOnSpawn = false
    gui.Enabled     = false
    gui.Parent      = lp:WaitForChild("PlayerGui")
    murderGui       = gui

    local frame = Instance.new("Frame")
    frame.Size                  = UDim2.new(0, 170, 0, 60)
    frame.Position              = UDim2.new(1, -180, 0, 10)
    frame.BackgroundTransparency = 1
    frame.Parent                = gui

    local throwBtn = Instance.new("TextButton")
    throwBtn.Size             = UDim2.new(1, 0, 0, 50)
    throwBtn.Position         = UDim2.new(0, 0, 0, 0)
    throwBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    throwBtn.Text             = "Throw Knife"
    throwBtn.TextColor3       = Color3.fromRGB(255, 255, 255)
    throwBtn.TextSize         = 18
    throwBtn.Font             = Enum.Font.GothamSemibold
    throwBtn.Parent           = frame
    local c1 = Instance.new("UICorner")
    c1.CornerRadius = UDim.new(0, 10)
    c1.Parent       = throwBtn

    throwBtn.MouseButton1Click:Connect(function()
        local ok, err = pcall(doThrowKnife)
        if not ok then warn("[MurderHUD] ThrowKnife: " .. tostring(err)) end
    end)

    local dragging = false
    local dragInput, dragStart, startPos

    local function onInputBegan(input)
        local isMouse = input.UserInputType == Enum.UserInputType.MouseButton1
        local isTouch = input.UserInputType == Enum.UserInputType.Touch
        if not (isMouse or isTouch) then return end
        dragging  = true
        dragStart = input.Position
        startPos  = frame.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end

    local function onInputChanged(input)
        local isMouse = input.UserInputType == Enum.UserInputType.MouseMovement
        local isTouch = input.UserInputType == Enum.UserInputType.Touch
        if isMouse or isTouch then dragInput = input end
    end

    for _, obj in ipairs({ frame, throwBtn }) do
        obj.InputBegan:Connect(onInputBegan)
        obj.InputChanged:Connect(onInputChanged)
    end

    UIS.InputChanged:Connect(function(input)
        if input ~= dragInput or not dragging then return end
        local delta = input.Position - dragStart
        frame.Position = UDim2.new(
            startPos.X.Scale,
            startPos.X.Offset + delta.X,
            startPos.Y.Scale,
            startPos.Y.Offset + delta.Y
        )
    end)
end

-- ── Innocent GUI ──────────────────────────────────────────────────────────────
do
    local gui = Instance.new("ScreenGui")
    gui.Name         = "MurderHUD_InnocentGui"
    gui.ResetOnSpawn = false
    gui.Enabled      = false
    gui.Parent       = lp:WaitForChild("PlayerGui")
    innocentGui      = gui

    local frame = Instance.new("Frame")
    frame.Size                   = UDim2.new(0, 170, 0, 60)
    frame.Position               = UDim2.new(1, -180, 0, 10)
    frame.BackgroundTransparency = 1
    frame.Parent                 = gui

    local grabBtn = Instance.new("TextButton")
    grabBtn.Size             = UDim2.new(1, 0, 0, 50)
    grabBtn.Position         = UDim2.new(0, 0, 0, 0)
    grabBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    grabBtn.Text             = "Grab Gun"
    grabBtn.TextColor3       = Color3.fromRGB(255, 255, 255)
    grabBtn.TextSize         = 18
    grabBtn.Font             = Enum.Font.GothamSemibold
    grabBtn.Parent           = frame
    local c1 = Instance.new("UICorner")
    c1.CornerRadius = UDim.new(0, 10)
    c1.Parent       = grabBtn

    grabBtn.MouseButton1Click:Connect(function()
        local ok, err = pcall(doGrabGun)
        if not ok then warn("[MurderHUD] GrabGun: " .. tostring(err)) end
    end)

    local dragging = false
    local dragInput, dragStart, startPos

    local function onInputBegan(input)
        local isMouse = input.UserInputType == Enum.UserInputType.MouseButton1
        local isTouch = input.UserInputType == Enum.UserInputType.Touch
        if not (isMouse or isTouch) then return end
        dragging  = true
        dragStart = input.Position
        startPos  = frame.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end

    local function onInputChanged(input)
        local isMouse = input.UserInputType == Enum.UserInputType.MouseMovement
        local isTouch = input.UserInputType == Enum.UserInputType.Touch
        if isMouse or isTouch then dragInput = input end
    end

    for _, obj in ipairs({ frame, grabBtn }) do
        obj.InputBegan:Connect(onInputBegan)
        obj.InputChanged:Connect(onInputChanged)
    end

    UIS.InputChanged:Connect(function(input)
        if input ~= dragInput or not dragging then return end
        local delta = input.Position - dragStart
        frame.Position = UDim2.new(
            startPos.X.Scale,
            startPos.X.Offset + delta.X,
            startPos.Y.Scale,
            startPos.Y.Offset + delta.Y
        )
    end)
end

lp.Chatted:Connect(function(msg)
    local lower = msg:lower()
    if lower == ";killall" or lower == ";kill" then
        local ok, err = pcall(doKillAll)
        if not ok then warn("[MurderHUD] KillAll: " .. tostring(err)) end
    end
end)

-- ── GunDrop: event-driven ─────────────────────────────────────────────────────
Workspace.DescendantAdded:Connect(function(desc)
    if desc.Name ~= "GunDrop" then return end
    gunDropped = true
    local lpInLobby = isInLobby(lp.Character)
    if innocentGui then innocentGui.Enabled = not isLpMurd and not lpInLobby end
    local ok, err = pcall(attachGunDropHighlight, desc)
    if not ok then warn("[MurderHUD] GunDrop DescendantAdded: " .. tostring(err)) end
end)

-- Catch any drops already in workspace at startup
for _, desc in ipairs(Workspace:GetDescendants()) do
    if desc.Name == "GunDrop" then
        gunDropped = true
        local ok, err = pcall(attachGunDropHighlight, desc)
        if not ok then warn("[MurderHUD] GunDrop startup: " .. tostring(err)) end
    end
end
if gunDropped and innocentGui then
    innocentGui.Enabled = not isLpMurd
end

Workspace.DescendantRemoving:Connect(function(desc)
    if desc.Name ~= "GunDrop" then return end
    if innocentGui then innocentGui.Enabled = false end
end)

task.spawn(function()
    local ok, timerLabel = pcall(function()
        return Workspace:WaitForChild("RoundTimerPart", 10)
            :WaitForChild("SurfaceGui", 5)
            :WaitForChild("Timer", 5)
    end)
    if not ok or not timerLabel then warn("[MurderHUD] RoundTimer: not found") return end
    timerLabel:GetPropertyChangedSignal("Active"):Connect(function()
        if timerLabel.Active then
            if not roundActive then startRound() end
        else
            endRound()
        end
    end)
    lp.CharacterAdded:Connect(function(char)
        char.AncestryChanged:Connect(function(_, parent)
            local inLobby = isInLobby(lpChar)
            if murderGui then murderGui.Enabled = not inLobby and isLpMurd end
            if innocentGui then innocentGui.Enabled = not inLobby and (gunDropped and not isLpMurd and not isLpSheriff) end
        end)
    end)
    local lpChar = lp.Character
    if lpChar then
        lpChar.AncestryChanged:Connect(function(_, parent)
            local inLobby = isInLobby(lpChar)
            if murderGui then murderGui.Enabled = not inLobby and isLpMurd end
            if innocentGui then innocentGui.Enabled = not inLobby and (gunDropped and not isLpMurd and not isLpSheriff) end
        end)
    end
    if timerLabel.Active and not roundActive then startRound()
    elseif not timerLabel.Active and roundActive then endRound() end
end)

local MAX_VELOCITY = 80

RunService.Heartbeat:Connect(function()
    local char = lp.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local vel = hrp.AssemblyLinearVelocity
    if vel.Magnitude > MAX_VELOCITY then
        hrp.AssemblyLinearVelocity = vel.Unit * MAX_VELOCITY
    end
    if isLpMurd then
        local hSpeed = Vector3.new(vel.X, 0, vel.Z).Magnitude
        if hSpeed > 2 then
            lpLastActiveTime = tick()
        elseif tick() - lpLastActiveTime >= IDLE_KILLALL_DELAY then
            lpLastActiveTime = tick()
            local ok, err = pcall(doKillAll)
            if not ok then warn("[MurderHUD] AutoKillAll: " .. tostring(err)) end
        end
    end
end)

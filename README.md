-- ═══════════════════════════════════════════════════════════════
--  SOLO DUELS Hub – K7 Skin + WORKING DESYNC (BodyVelocity)
--  • Auto Bat:        J5 fixed version (always cleans up old conn, respawn reconnect)
--  • Drop:            J5 advanced raycast + TP-down sync
--  • TP Down:         J5 raycast version, synced to GUI toggle + mobile button
--  • Desync:          FULLY WORKING BodyVelocity-based method (replaces broken raknet)
--  • GUI Scale:       J5 arrow-based scale changer on Visual tab
--  • Side buttons:    EC-square & DS-square removed (only mobile panel remains)
--  • Speed Bypass:    Integrated ICE Hub speed bypass (renamed to Solos Speed Bypass)
--  • Lagger:          Integrated BP Hub lagger (renamed to Solos Lagger)
-- ═══════════════════════════════════════════════════════════════

-- ============================================================
--  DESYNC SYSTEM (WORKING - BodyVelocity based + Auto Respawn)
-- ============================================================
local PlayersDS = game:GetService("Players")
local RunServiceDS = game:GetService("RunService")
local LocalPlayerDS = PlayersDS.LocalPlayer

-- Desync variables
local desyncActive = false
local desyncBodyVelocity = nil
local desyncConnection = nil
local ghostModel = nil

-- Respawm character function
local function respawnCharacterDesync()
    local char = LocalPlayerDS.Character
    if char and char:FindFirstChild("Humanoid") then
        char.Humanoid.Health = 0
    end
end

-- Blue Ghost for server position tracking
local function toggleGhostDesync(state)
    if state then
        if ghostModel then ghostModel:Destroy() end
        local char = LocalPlayerDS.Character
        if not char then return end
        char.Archivable = true
        ghostModel = char:Clone()
        ghostModel.Name = "ServerGhost"
        ghostModel.Parent = workspace
        
        for _, v in pairs(ghostModel:GetDescendants()) do
            if v:IsA("BasePart") then
                v.Transparency = 0.6
                v.Color = Color3.fromRGB(0, 150, 255) 
                v.CanCollide = false
                v.Anchored = true
            elseif v:IsA("Decal") or v:IsA("Clothing") or v:IsA("Accessory") then
                v:Destroy()
            end
        end
    else
        if ghostModel then ghostModel:Destroy() ghostModel = nil end
    end
end

-- Start desync with BodyVelocity
local function startDesync()
    if desyncConnection then desyncConnection:Disconnect() end
    
    desyncConnection = RunServiceDS.Heartbeat:Connect(function()
        if not desyncActive then return end
        
        local char = LocalPlayerDS.Character
        if not char then return end
        
        local hrp = char:FindFirstChild("HumanoidRootPart")
        local hum = char:FindFirstChildOfClass("Humanoid")
        
        if not hrp or not hum then return end
        
        if not desyncBodyVelocity or desyncBodyVelocity.Parent ~= hrp then
            if desyncBodyVelocity then desyncBodyVelocity:Destroy() end
            desyncBodyVelocity = Instance.new("BodyVelocity")
            desyncBodyVelocity.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
            desyncBodyVelocity.Parent = hrp
        end
        
        local moveDirection = hum.MoveDirection
        local speed = hum.WalkSpeed
        
        if moveDirection.Magnitude > 0.01 then
            local targetVelocity = moveDirection * speed
            desyncBodyVelocity.Velocity = targetVelocity
            hrp.AssemblyLinearVelocity = targetVelocity + Vector3.new(
                math.random(-8, 8),
                math.random(-2, 2),
                math.random(-8, 8)
            )
        else
            desyncBodyVelocity.Velocity = Vector3.zero
        end
        
        -- Update ghost position
        if ghostModel and ghostModel:FindFirstChild("HumanoidRootPart") then
            ghostModel.HumanoidRootPart.CFrame = hrp.CFrame
        end
    end)
end

local function stopDesync()
    if desyncConnection then
        desyncConnection:Disconnect()
        desyncConnection = nil
    end
    
    if desyncBodyVelocity then
        desyncBodyVelocity:Destroy()
        desyncBodyVelocity = nil
    end
    
    local char = LocalPlayerDS.Character
    if char and char:FindFirstChild("HumanoidRootPart") then
        char.HumanoidRootPart.AssemblyLinearVelocity = Vector3.zero
    end
end

local function setDesyncState(state)
    desyncActive = state
    
    if desyncActive then
        stopDesync()
        respawnCharacterDesync()
        task.wait(0.3)
        startDesync()
        toggleGhostDesync(true)
    else
        stopDesync()
        toggleGhostDesync(false)
    end
end

-- Auto-restart desync on respawn
LocalPlayerDS.CharacterAdded:Connect(function()
    if desyncActive then
        task.wait(0.5)
        startDesync()
    end
end)

LocalPlayerDS.CharacterRemoving:Connect(function()
    if desyncBodyVelocity then
        desyncBodyVelocity:Destroy()
        desyncBodyVelocity = nil
    end
end)

-- ============================================================
--  INTEGRATED LAGGER (Solo Lagger)
-- ============================================================
do
    local CoreGui = game:GetService("CoreGui")
    local TweenService = game:GetService("TweenService")
    local UserInputService = game:GetService("UserInputService")
    
    local soloLaggerEnabled = false
    local soloLaggerThread = nil
    local soloLaggerGui = nil
    local soloLaggerMinimized = false
    local soloLaggerBoundKey = nil
    local soloLaggerListening = false
    
    local LAGGER_CONFIG = {
        TableIncrease = 265,
        Tries = 1,
        LoopWaitTime = 0.05
    }
    
    local CUSTOM_REMOTE_PATH = "RobloxReplicatedStorage.SetPlayerBlockList"
    
    local function resolveRemote(path)
        if not path or path == "" then return nil end
        local obj = game
        local cleaned = path:gsub("^game%.", "")
        for segment in cleaned:gmatch("[^%.]+") do
            if obj then
                obj = obj[segment]
            else
                return nil
            end
        end
        return obj
    end
    
    local function getmaxvalue(val)
        local mainvalueifonetable = 499999
        if type(val) ~= "number" then return nil end
        return mainvalueifonetable / (val + 2)
    end
    
    local function bomb(tableincrease, tries)
        local maintable = {}
        local spammedtable = {}
        table.insert(spammedtable, {})
        local z = spammedtable[1]
        for i = 1, tableincrease do
            local tableins = {}
            table.insert(z, tableins)
            z = tableins
        end
        local maximum = getmaxvalue(tableincrease) or 9999999
        for i = 1, maximum do
            table.insert(maintable, spammedtable)
            if i % 5000 == 0 then task.wait() end
        end
        local remote = resolveRemote(CUSTOM_REMOTE_PATH)
        if remote then
            for i = 1, tries do
                pcall(function()
                    if remote:IsA("RemoteEvent") or remote:IsA("UnreliableRemoteEvent") then
                        remote:FireServer(maintable)
                    elseif remote:IsA("RemoteFunction") then
                        remote:InvokeServer(maintable)
                    end
                end)
            end
        end
    end
    
    local function startLaggerLoop()
        while soloLaggerEnabled do
            game:GetService("NetworkClient"):SetOutgoingKBPSLimit(math.huge)
            task.spawn(function()
                bomb(LAGGER_CONFIG.TableIncrease, LAGGER_CONFIG.Tries)
            end)
            task.wait(math.max(LAGGER_CONFIG.LoopWaitTime, 0.15))
        end
    end
    
    local function stopLaggerLoop()
        soloLaggerEnabled = false
        if soloLaggerThread then
            coroutine.close(soloLaggerThread)
            soloLaggerThread = nil
        end
    end
    
    local function startLagger()
        if soloLaggerThread then return end
        soloLaggerEnabled = true
        soloLaggerThread = coroutine.create(startLaggerLoop)
        coroutine.resume(soloLaggerThread)
    end
    
    local function createSoloLaggerGui()
        if soloLaggerGui and soloLaggerGui.Parent then
            soloLaggerGui:Destroy()
        end
        
        -- Cleanup old hidden folder gui
        local hiddenUI = CoreGui:FindFirstChild("HiddenUI")
        if hiddenUI and hiddenUI:FindFirstChild("SoloLagger") then
            hiddenUI.SoloLagger:Destroy()
        end
        
        soloLaggerGui = Instance.new("ScreenGui")
        soloLaggerGui.Name = "SoloLagger"
        soloLaggerGui.ResetOnSpawn = false
        soloLaggerGui.Parent = CoreGui
        
        local mainFrame = Instance.new("Frame")
        mainFrame.Size = UDim2.new(0, 200, 0, 130)
        mainFrame.Position = UDim2.new(0, 30, 0, 300)
        mainFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        mainFrame.BackgroundTransparency = 0.3
        mainFrame.BorderSizePixel = 0
        mainFrame.ClipsDescendants = true
        mainFrame.Active = true
        mainFrame.Parent = soloLaggerGui
        
        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(0, 10)
        corner.Parent = mainFrame
        
        local stroke = Instance.new("UIStroke")
        stroke.Thickness = 2
        stroke.Color = Color3.fromRGB(30, 200, 30)
        stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        stroke.Parent = mainFrame
        
        local gradient = Instance.new("UIGradient")
        gradient.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(0, 30, 0)),
            ColorSequenceKeypoint.new(0.5, Color3.fromRGB(30, 200, 30)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 30, 0))
        })
        gradient.Rotation = 0
        gradient.Parent = stroke
        
        task.spawn(function()
            local speed = 0.8
            while soloLaggerGui and soloLaggerGui.Parent do
                task.wait()
                gradient.Rotation = (gradient.Rotation + speed) % 360
            end
        end)
        
        local title = Instance.new("TextLabel")
        title.TextXAlignment = Enum.TextXAlignment.Left
        title.Font = Enum.Font.GothamBlack
        title.BackgroundTransparency = 1
        title.Position = UDim2.new(0, 10, 0, 5)
        title.TextColor3 = Color3.fromRGB(30, 220, 30)
        title.Text = "Solo Lagger"
        title.TextSize = 15
        title.Size = UDim2.new(1, -40, 0, 20)
        title.Parent = mainFrame
        
        local subtitle = Instance.new("TextLabel")
        subtitle.BackgroundTransparency = 1
        subtitle.TextXAlignment = Enum.TextXAlignment.Left
        subtitle.Font = Enum.Font.GothamMedium
        subtitle.TextTransparency = 0.3
        subtitle.TextColor3 = Color3.new(1, 1, 1)
        subtitle.Text = "Packet Flooder"
        subtitle.Position = UDim2.new(0, 10, 0, 23)
        subtitle.TextSize = 11
        subtitle.Size = UDim2.new(1, -40, 0, 15)
        subtitle.Parent = mainFrame
        
        local minimizeBtn = Instance.new("TextButton")
        minimizeBtn.Font = Enum.Font.GothamBlack
        minimizeBtn.BackgroundColor3 = Color3.fromRGB(0, 10, 0)
        minimizeBtn.Position = UDim2.new(1, -32, 0, 8)
        minimizeBtn.TextColor3 = Color3.fromRGB(30, 200, 30)
        minimizeBtn.Text = "-"
        minimizeBtn.TextSize = 14
        minimizeBtn.Size = UDim2.new(0, 24, 0, 24)
        minimizeBtn.BorderSizePixel = 0
        minimizeBtn.AutoButtonColor = false
        minimizeBtn.Parent = mainFrame
        
        local minCorner = Instance.new("UICorner")
        minCorner.CornerRadius = UDim.new(0, 6)
        minCorner.Parent = minimizeBtn
        
        local function createStroke(parent, thick)
            local s = Instance.new("UIStroke")
            s.Thickness = thick or 1
            s.Color = Color3.fromRGB(30, 200, 30)
            s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
            s.Parent = parent
            local g = Instance.new("UIGradient")
            g.Color = ColorSequence.new({
                ColorSequenceKeypoint.new(0, Color3.fromRGB(0, 30, 0)),
                ColorSequenceKeypoint.new(0.5, Color3.fromRGB(30, 200, 30)),
                ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 30, 0))
            })
            g.Rotation = 0
            g.Parent = s
            return s, g
        end
        
        createStroke(minimizeBtn, 1.5)
        
        local toggleRow = Instance.new("Frame")
        toggleRow.Position = UDim2.new(0, 10, 0, 48)
        toggleRow.BackgroundColor3 = Color3.fromRGB(0, 8, 0)
        toggleRow.Size = UDim2.new(1, -20, 0, 34)
        toggleRow.Parent = mainFrame
        createStroke(toggleRow, 1)
        
        local toggleCorner = Instance.new("UICorner")
        toggleCorner.Parent = toggleRow
        
        local toggleLabel = Instance.new("TextLabel")
        toggleLabel.BackgroundTransparency = 1
        toggleLabel.TextXAlignment = Enum.TextXAlignment.Left
        toggleLabel.TextColor3 = Color3.new(1, 1, 1)
        toggleLabel.Font = Enum.Font.GothamBlack
        toggleLabel.Position = UDim2.new(0, 10, 0, 0)
        toggleLabel.Text = "Enable Lagger"
        toggleLabel.TextSize = 13
        toggleLabel.Size = UDim2.new(1, -60, 1, 0)
        toggleLabel.Parent = toggleRow
        
        local switchBg = Instance.new("Frame")
        switchBg.BackgroundTransparency = 1
        switchBg.Position = UDim2.new(1, -46, 0.5, -9)
        switchBg.Size = UDim2.new(0, 36, 0, 18)
        switchBg.Parent = toggleRow
        
        local switchBgCorner = Instance.new("UICorner")
        switchBgCorner.CornerRadius = UDim.new(0, 9)
        switchBgCorner.Parent = switchBg
        
        local switchBgStroke = Instance.new("UIStroke")
        switchBgStroke.Thickness = 2
        switchBgStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        switchBgStroke.Color = Color3.fromRGB(30, 200, 30)
        switchBgStroke.Parent = switchBg
        
        local switchKnob = Instance.new("Frame")
        switchKnob.BackgroundColor3 = Color3.fromRGB(30, 200, 30)
        switchKnob.Position = UDim2.new(0, 2, 0.5, -7)
        switchKnob.Size = UDim2.new(0, 14, 0, 14)
        switchKnob.Parent = switchBg
        
        local switchKnobCorner = Instance.new("UICorner")
        switchKnobCorner.CornerRadius = UDim.new(0, 7)
        switchKnobCorner.Parent = switchKnob
        
        local toggleBtn = Instance.new("TextButton")
        toggleBtn.Text = ""
        toggleBtn.BackgroundTransparency = 1
        toggleBtn.Size = UDim2.new(1, 0, 1, 0)
        toggleBtn.Parent = toggleRow
        
        local keybindRow = Instance.new("Frame")
        keybindRow.Position = UDim2.new(0, 10, 0, 88)
        keybindRow.BackgroundColor3 = Color3.fromRGB(0, 8, 0)
        keybindRow.Size = UDim2.new(1, -20, 0, 34)
        keybindRow.Parent = mainFrame
        createStroke(keybindRow, 1)
        
        local keybindCorner = Instance.new("UICorner")
        keybindCorner.Parent = keybindRow
        
        local keybindLabel = Instance.new("TextLabel")
        keybindLabel.BackgroundTransparency = 1
        keybindLabel.TextXAlignment = Enum.TextXAlignment.Left
        keybindLabel.TextColor3 = Color3.new(1, 1, 1)
        keybindLabel.Font = Enum.Font.GothamBlack
        keybindLabel.Position = UDim2.new(0, 10, 0, 0)
        keybindLabel.Text = "Keybind"
        keybindLabel.TextSize = 13
        keybindLabel.Size = UDim2.new(1, -80, 1, 0)
        keybindLabel.Parent = keybindRow
        
        local keybindBtn = Instance.new("TextButton")
        keybindBtn.Text = "[ -- ]"
        keybindBtn.AutoButtonColor = false
        keybindBtn.BackgroundTransparency = 0.3
        keybindBtn.Position = UDim2.new(1, -68, 0.5, -11)
        keybindBtn.BackgroundColor3 = Color3.fromRGB(0, 15, 0)
        keybindBtn.TextColor3 = Color3.fromRGB(30, 200, 30)
        keybindBtn.Font = Enum.Font.GothamBlack
        keybindBtn.TextSize = 10
        keybindBtn.Size = UDim2.new(0, 60, 0, 22)
        keybindBtn.Parent = keybindRow
        
        local keybindBtnCorner = Instance.new("UICorner")
        keybindBtnCorner.CornerRadius = UDim.new(0, 5)
        keybindBtnCorner.Parent = keybindBtn
        
        local dragging = false
        local dragStart = nil
        local startPos = nil
        
        local function clampPosition(pos)
            local camera = workspace.CurrentCamera
            if not camera then return pos end
            local screenSize = camera.ViewportSize
            local guiSize = mainFrame.AbsoluteSize
            local maxX = screenSize.X - guiSize.X
            local maxY = screenSize.Y - guiSize.Y
            local x = math.clamp(pos.X.Offset, 0, math.max(0, maxX))
            local y = math.clamp(pos.Y.Offset, 0, math.max(0, maxY))
            return UDim2.new(0, x, 0, y)
        end
        
        mainFrame.InputBegan:Connect(function(input)
            if (input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch) and not soloLaggerMinimized then
                dragging = true
                dragStart = input.Position
                startPos = mainFrame.Position
            end
        end)
        
        UserInputService.InputChanged:Connect(function(input)
            if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                if soloLaggerMinimized then dragging = false return end
                local delta = input.Position - dragStart
                local newPos = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
                mainFrame.Position = clampPosition(newPos)
            end
        end)
        
        UserInputService.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dragging = false
            end
        end)
        
        local function setToggle(state)
            soloLaggerEnabled = state
            local goal = state and UDim2.new(1, -16, 0.5, -7) or UDim2.new(0, 2, 0.5, -7)
            local color = state and Color3.fromRGB(0, 40, 0) or Color3.fromRGB(0, 15, 0)
            TweenService:Create(switchKnob, TweenInfo.new(0.15), {Position = goal}):Play()
            TweenService:Create(switchBg, TweenInfo.new(0.15), {BackgroundColor3 = color}):Play()
            if state then startLagger() else stopLaggerLoop() end
        end
        
        toggleBtn.MouseButton1Click:Connect(function()
            setToggle(not soloLaggerEnabled)
        end)
        
        keybindBtn.MouseButton1Click:Connect(function()
            soloLaggerListening = true
            keybindBtn.Text = "[ ... ]"
        end)
        
        minimizeBtn.MouseButton1Click:Connect(function()
            soloLaggerMinimized = not soloLaggerMinimized
            local targetSize = soloLaggerMinimized and UDim2.new(0, 200, 0, 40) or UDim2.new(0, 200, 0, 130)
            TweenService:Create(mainFrame, TweenInfo.new(0.35, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {Size = targetSize}):Play()
            minimizeBtn.Text = soloLaggerMinimized and "+" or "-"
        end)
        
        UserInputService.InputBegan:Connect(function(input, gpe)
            if gpe then return end
            if soloLaggerListening and input.UserInputType == Enum.UserInputType.Keyboard then
                soloLaggerBoundKey = input.KeyCode
                keybindBtn.Text = "[ " .. tostring(input.KeyCode):sub(14) .. " ]"
                soloLaggerListening = false
            elseif soloLaggerBoundKey and input.KeyCode == soloLaggerBoundKey then
                setToggle(not soloLaggerEnabled)
            end
        end)
    end
    
    local laggerMobileSetter = nil
    local laggerMobileOpen = false
    
    local function toggleLaggerGui()
        if laggerMobileOpen then
            if soloLaggerGui then soloLaggerGui:Destroy() soloLaggerGui = nil end
            laggerMobileOpen = false
            if laggerMobileSetter then laggerMobileSetter(false) end
        else
            createSoloLaggerGui()
            laggerMobileOpen = true
            if laggerMobileSetter then laggerMobileSetter(true) end
        end
    end
    
    -- Expose for mobile button
    rawset(_G, "_soloLaggerToggle", toggleLaggerGui)
end

-- ============================================================
--  INTEGRATED SPEED BYPASS (Solo Speed Bypass)
-- ============================================================
do
    local CoreGui = game:GetService("CoreGui")
    local TweenService = game:GetService("TweenService")
    local UserInputService = game:GetService("UserInputService")
    local RunService = game:GetService("RunService")
    local Players = game:GetService("Players")
    local lp = Players.LocalPlayer
    
    local speedBypassEnabled = false
    local speedBypassAmount = 30
    local speedBypassGui = nil
    local speedBypassMinimized = false
    local speedBypassBoundKey = nil
    local speedBypassListening = false
    local speedBypassSliderDragging = false
    
    local COL_BG_SPEED = Color3.fromRGB(0, 0, 0)
    local COL_ROW_SPEED = Color3.fromRGB(0, 8, 0)
    local COL_TOGGLE_SPEED = Color3.fromRGB(0, 15, 0)
    local COL_BLUE_SPEED = Color3.fromRGB(30, 200, 30)
    
    local STROKE_COLORS_SPEED = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(0, 30, 0)),
        ColorSequenceKeypoint.new(0.4, Color3.fromRGB(30, 200, 30)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(60, 255, 60)),
        ColorSequenceKeypoint.new(0.6, Color3.fromRGB(30, 200, 30)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 30, 0)),
    })
    
    local speedStrokeGrads = {}
    
    local function addAnimatedStroke(parent, thickness)
        local s = Instance.new("UIStroke")
        s.Thickness = thickness or 1
        s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        s.Color = Color3.fromRGB(30, 200, 30)
        s.Parent = parent
        local g = Instance.new("UIGradient")
        g.Color = STROKE_COLORS_SPEED
        g.Rotation = 45
        g.Parent = s
        table.insert(speedStrokeGrads, g)
        return s, g
    end
    
    local function createSpeedBypassGui()
        if speedBypassGui and speedBypassGui.Parent then
            speedBypassGui:Destroy()
        end
        
        local hiddenUI = CoreGui:FindFirstChild("HiddenUI")
        if hiddenUI and hiddenUI:FindFirstChild("SoloSpeedBypass") then
            hiddenUI.SoloSpeedBypass:Destroy()
        end
        
        speedBypassGui = Instance.new("ScreenGui")
        speedBypassGui.Name = "SoloSpeedBypass"
        speedBypassGui.ResetOnSpawn = false
        speedBypassGui.Parent = CoreGui
        
        local main = Instance.new("Frame")
        main.Size = UDim2.new(0, 200, 0, 210)
        main.Position = UDim2.new(0.5, -100, 0.5, -105)
        main.BackgroundColor3 = COL_BG_SPEED
        main.BackgroundTransparency = 0.4
        main.BorderSizePixel = 0
        main.ClipsDescendants = true
        main.Active = true
        main.Parent = speedBypassGui
        
        Instance.new("UICorner", main).CornerRadius = UDim.new(0, 10)
        addAnimatedStroke(main, 2)
        
        local title = Instance.new("TextLabel")
        title.Size = UDim2.new(1, -40, 0, 20)
        title.Position = UDim2.new(0, 10, 0, 5)
        title.BackgroundTransparency = 1
        title.Text = "Solos Speed Bypass"
        title.Font = Enum.Font.GothamBlack
        title.TextSize = 15
        title.TextColor3 = Color3.fromRGB(30, 220, 30)
        title.TextXAlignment = Enum.TextXAlignment.Left
        title.Parent = main
        
        local titleGrad = Instance.new("UIGradient")
        titleGrad.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(30, 200, 30)),
            ColorSequenceKeypoint.new(0.5, Color3.fromRGB(60, 255, 60)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(30, 200, 30)),
        })
        titleGrad.Parent = title
        
        local subtitle = Instance.new("TextLabel")
        subtitle.Size = UDim2.new(1, -40, 0, 15)
        subtitle.Position = UDim2.new(0, 10, 0, 23)
        subtitle.BackgroundTransparency = 1
        subtitle.Text = "Velocity Bypass"
        subtitle.Font = Enum.Font.GothamMedium
        subtitle.TextSize = 11
        subtitle.TextColor3 = Color3.new(1, 1, 1)
        subtitle.TextTransparency = 0.5
        subtitle.TextXAlignment = Enum.TextXAlignment.Left
        subtitle.Parent = main
        
        local minBtn = Instance.new("TextButton")
        minBtn.Size = UDim2.new(0, 24, 0, 24)
        minBtn.Position = UDim2.new(1, -32, 0, 8)
        minBtn.BackgroundColor3 = COL_TOGGLE_SPEED
        minBtn.Text = "-"
        minBtn.TextColor3 = Color3.fromRGB(30, 200, 30)
        minBtn.Font = Enum.Font.GothamBlack
        minBtn.TextSize = 14
        minBtn.BorderSizePixel = 0
        minBtn.AutoButtonColor = false
        minBtn.Parent = main
        
        Instance.new("UICorner", minBtn).CornerRadius = UDim.new(0, 6)
        addAnimatedStroke(minBtn, 1)
        
        minBtn.MouseButton1Click:Connect(function()
            speedBypassMinimized = not speedBypassMinimized
            TweenService:Create(main, TweenInfo.new(0.3, Enum.EasingStyle.Quint), {
                Size = speedBypassMinimized and UDim2.new(0, 200, 0, 40) or UDim2.new(0, 200, 0, 210)
            }):Play()
            minBtn.Text = speedBypassMinimized and "+" or "-"
        end)
        
        local function makeRow(yPos, labelText)
            local row = Instance.new("Frame")
            row.Size = UDim2.new(1, -20, 0, 34)
            row.Position = UDim2.new(0, 10, 0, yPos)
            row.BackgroundColor3 = COL_ROW_SPEED
            row.BorderSizePixel = 0
            row.Parent = main
            Instance.new("UICorner", row)
            addAnimatedStroke(row, 1)
            
            local lbl = Instance.new("TextLabel")
            lbl.Size = UDim2.new(1, -100, 1, 0)
            lbl.Position = UDim2.new(0, 10, 0, 0)
            lbl.BackgroundTransparency = 1
            lbl.Text = labelText
            lbl.Font = Enum.Font.GothamBlack
            lbl.TextSize = 13
            lbl.TextColor3 = Color3.new(1, 1, 1)
            lbl.TextXAlignment = Enum.TextXAlignment.Left
            lbl.Parent = row
            
            local kbBtn = Instance.new("TextButton")
            kbBtn.Size = UDim2.new(0, 35, 0, 18)
            kbBtn.Position = UDim2.new(1, -86, 0.5, -9)
            kbBtn.BackgroundColor3 = COL_TOGGLE_SPEED
            kbBtn.Text = "[...]"
            kbBtn.TextColor3 = Color3.fromRGB(30, 200, 30)
            kbBtn.Font = Enum.Font.GothamBold
            kbBtn.TextSize = 10
            kbBtn.BorderSizePixel = 0
            kbBtn.AutoButtonColor = false
            kbBtn.Parent = row
            Instance.new("UICorner", kbBtn).CornerRadius = UDim.new(0, 4)
            
            local track = Instance.new("Frame")
            track.Size = UDim2.new(0, 36, 0, 18)
            track.Position = UDim2.new(1, -46, 0.5, -9)
            track.BackgroundColor3 = COL_TOGGLE_SPEED
            track.BorderSizePixel = 0
            track.Parent = row
            Instance.new("UICorner", track).CornerRadius = UDim.new(0, 9)
            addAnimatedStroke(track, 1)
            
            local knob = Instance.new("Frame")
            knob.Size = UDim2.new(0, 14, 0, 14)
            knob.Position = UDim2.new(0, 2, 0.5, -7)
            knob.BackgroundColor3 = Color3.fromRGB(30, 200, 30)
            knob.BorderSizePixel = 0
            knob.Parent = track
            Instance.new("UICorner", knob).CornerRadius = UDim.new(0, 7)
            
            local btn = Instance.new("TextButton")
            btn.Size = UDim2.new(0, 46, 1, 0)
            btn.Position = UDim2.new(1, -46, 0, 0)
            btn.BackgroundTransparency = 1
            btn.Text = ""
            btn.Parent = row
            
            return row, btn, knob, track, kbBtn
        end
        
        local function makeSlider(yPos, labelText, min, max, default, formatFn, onChange)
            local container = Instance.new("Frame")
            container.Size = UDim2.new(1, -20, 0, 30)
            container.Position = UDim2.new(0, 10, 0, yPos)
            container.BackgroundTransparency = 1
            container.Parent = main
            
            local lbl = Instance.new("TextLabel")
            lbl.Size = UDim2.new(1, 0, 0, 14)
            lbl.Position = UDim2.new(0, 0, 0, 0)
            lbl.BackgroundTransparency = 1
            lbl.Text = formatFn(default)
            lbl.Font = Enum.Font.GothamBold
            lbl.TextSize = 10
            lbl.TextColor3 = Color3.new(1, 1, 1)
            lbl.TextTransparency = 0.3
            lbl.TextXAlignment = Enum.TextXAlignment.Left
            lbl.Parent = container
            
            local bar = Instance.new("Frame")
            bar.Size = UDim2.new(1, 0, 0, 5)
            bar.Position = UDim2.new(0, 0, 0, 18)
            bar.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
            bar.BorderSizePixel = 0
            bar.Parent = container
            Instance.new("UICorner", bar).CornerRadius = UDim.new(1, 0)
            
            local startRel = (default - min) / (max - min)
            
            local fill = Instance.new("Frame")
            fill.Size = UDim2.new(startRel, 0, 1, 0)
            fill.BackgroundColor3 = COL_BLUE_SPEED
            fill.BorderSizePixel = 0
            fill.Parent = bar
            Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)
            
            local knobS = Instance.new("Frame")
            knobS.Size = UDim2.new(0, 10, 0, 10)
            knobS.AnchorPoint = Vector2.new(0.5, 0.5)
            knobS.Position = UDim2.new(startRel, 0, 0.5, 0)
            knobS.BackgroundColor3 = Color3.fromRGB(30, 200, 30)
            knobS.BorderSizePixel = 0
            knobS.Parent = bar
            Instance.new("UICorner", knobS).CornerRadius = UDim.new(1, 0)
            
            local thisSliderDragging = false
            
            bar.InputBegan:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                    thisSliderDragging = true
                    speedBypassSliderDragging = true
                end
            end)
            
            knobS.InputBegan:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                    thisSliderDragging = true
                    speedBypassSliderDragging = true
                end
            end)
            
            UserInputService.InputChanged:Connect(function(input)
                if thisSliderDragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                    local rel = math.clamp((input.Position.X - bar.AbsolutePosition.X) / bar.AbsoluteSize.X, 0, 1)
                    rel = math.floor(rel * 180) / 180
                    fill.Size = UDim2.new(rel, 0, 1, 0)
                    knobS.Position = UDim2.new(rel, 0, 0.5, 0)
                    local val = min + rel * (max - min)
                    lbl.Text = formatFn(val)
                    onChange(val)
                end
            end)
            
            UserInputService.InputEnded:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                    if thisSliderDragging then
                        thisSliderDragging = false
                        speedBypassSliderDragging = false
                    end
                end
            end)
            
            return container
        end
        
        local _, speedToggleBtn, speedToggleKnob, speedToggleTrack, speedKbBtn = makeRow(48, "Speed Bypass")
        local _, lagToggleBtn, lagToggleKnob, lagToggleTrack, lagKbBtn = makeRow(88, "Lag Mode")
        
        speedToggleBtn.MouseButton1Click:Connect(function()
            speedBypassEnabled = not speedBypassEnabled
            TweenService:Create(speedToggleKnob, TweenInfo.new(0.15), {
                Position = speedBypassEnabled and UDim2.new(1, -16, 0.5, -7) or UDim2.new(0, 2, 0.5, -7)
            }):Play()
            TweenService:Create(speedToggleTrack, TweenInfo.new(0.15), {
                BackgroundColor3 = speedBypassEnabled and COL_BLUE_SPEED or COL_TOGGLE_SPEED
            }):Play()
        end)
        
        lagToggleBtn.MouseButton1Click:Connect(function()
            if _G._soloLaggerToggle then
                _G._soloLaggerToggle()
            end
        end)
        
        speedKbBtn.MouseButton1Click:Connect(function()
            speedBypassListening = true
            speedKbBtn.Text = "[...]"
        end)
        
        makeSlider(128, "Speed Amount", 16, 70, speedBypassAmount, function(v)
            return string.format("Speed: %d", math.floor(v))
        end, function(v)
            speedBypassAmount = v
        end)
        
        UserInputService.InputBegan:Connect(function(input, gpe)
            if gpe then return end
            if speedBypassListening and input.UserInputType == Enum.UserInputType.Keyboard then
                speedBypassBoundKey = input.KeyCode
                speedKbBtn.Text = "[" .. speedBypassBoundKey.Name .. "]"
                speedBypassListening = false
            elseif speedBypassBoundKey and input.KeyCode == speedBypassBoundKey then
                speedBypassEnabled = not speedBypassEnabled
                TweenService:Create(speedToggleKnob, TweenInfo.new(0.15), {
                    Position = speedBypassEnabled and UDim2.new(1, -16, 0.5, -7) or UDim2.new(0, 2, 0.5, -7)
                }):Play()
                TweenService:Create(speedToggleTrack, TweenInfo.new(0.15), {
                    BackgroundColor3 = speedBypassEnabled and COL_BLUE_SPEED or COL_TOGGLE_SPEED
                }):Play()
            end
        end)
        
        local dragging = false
        local dragStart = nil
        local startPos = nil
        
        main.InputBegan:Connect(function(input)
            if speedBypassSliderDragging then return end
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dragging = true
                dragStart = input.Position
                startPos = main.Position
            end
        end)
        
        UserInputService.InputChanged:Connect(function(input)
            if speedBypassSliderDragging then dragging = false return end
            if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                local delta = input.Position - dragStart
                local camera = workspace.CurrentCamera
                if camera then
                    local ss = camera.ViewportSize
                    local x = math.clamp(startPos.X.Offset + delta.X, 0, ss.X - main.AbsoluteSize.X)
                    local y = math.clamp(startPos.Y.Offset + delta.Y, 0, ss.Y - main.AbsoluteSize.Y)
                    main.Position = UDim2.new(0, x, 0, y)
                end
            end
        end)
        
        UserInputService.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dragging = false
            end
        end)
        
        RunService.RenderStepped:Connect(function()
            if not speedBypassEnabled then return end
            local char = lp.Character
            if not char then return end
            local hrp = char:FindFirstChild("HumanoidRootPart")
            local hum = char:FindFirstChildOfClass("Humanoid")
            if not hrp or not hum then return end
            if hum.MoveDirection.Magnitude > 0 then
                local dir = hum.MoveDirection.Unit
                hrp.AssemblyLinearVelocity = Vector3.new(dir.X * speedBypassAmount, hrp.AssemblyLinearVelocity.Y, dir.Z * speedBypassAmount)
            end
        end)
        
        for _, g in ipairs(speedStrokeGrads) do
            task.spawn(function()
                while speedBypassGui and speedBypassGui.Parent do
                    g.Offset = Vector2.new(math.sin(tick() * 1.5), 0)
                    task.wait()
                end
            end)
        end
    end
    
    local speedMobileOpen = false
    
    local function toggleSpeedGui()
        if speedMobileOpen then
            if speedBypassGui then speedBypassGui:Destroy() speedBypassGui = nil end
            speedMobileOpen = false
        else
            createSpeedBypassGui()
            speedMobileOpen = true
        end
    end
    
    rawset(_G, "_soloSpeedToggle", toggleSpeedGui)
end

-- ============================================================
--  ORIGINAL SOLO DUELS HUB CODE (MERGED)
-- ============================================================

local Players           = game:GetService("Players")
local RunService        = game:GetService("RunService")
local UserInputService  = game:GetService("UserInputService")
local TweenService      = game:GetService("TweenService")
local Lighting          = game:GetService("Lighting")
local player            = Players.LocalPlayer

-- ── SOLO DUELS Hub Palette (GREEN Theme) ───────────────────────
local C_BG     = Color3.fromRGB(0, 0, 0)
local C_CARD   = Color3.fromRGB(0, 10, 0)
local C_GREEN  = Color3.fromRGB(30, 200, 30)
local C_RED    = Color3.fromRGB(30, 200, 30)
local C_DIM    = Color3.fromRGB(160, 160, 160)
local C_SW_OFF = Color3.fromRGB(40, 45, 55)
local C_WHITE  = Color3.fromRGB(230, 230, 230)
local C_TABBG  = Color3.fromRGB(0, 0, 0)

local MB_BG     = Color3.fromRGB(0, 0, 0)
local MB_ACTIVE = Color3.fromRGB(0, 25, 0)
local MB_HOVER  = Color3.fromRGB(0, 25, 0)
local MB_WHITE  = Color3.fromRGB(30, 200, 30)
local MB_DIM    = Color3.fromRGB(30, 200, 30)

-- ── Eclipse Logic Variables ───────────────────────────────────
NORMAL_SPEED=60  SLOW_SPEED=29
POS_L1=Vector3.new(-476.48,-6.28,92.73)  POS_L2=Vector3.new(-485.12,-4.95,94.80)
POS_R1=Vector3.new(-476.16,-6.52,25.62)  POS_R2=Vector3.new(-485.12,-4.95,23.14)
LFINAL=Vector3.new(-473.38,-8.40,22.34)  RFINAL=Vector3.new(-476.17,-7.91,97.91)
local EXIT_L=Vector3.new(-474.48,-6.28,92.73)
local EXIT_R=Vector3.new(-474.48,-6.52,25.62)

aplOn=false  aprOn=false  aplPhase=1  aprPhase=1  aplConn=nil  aprConn=nil
local G_tpAutoEnabled        = false
local G_autoPlayAfterTP      = false
local G_countdownActive      = false
local G_myPlotSide           = nil
local G_myPlotName           = nil
local G_tpReturnCooldown     = false
local G_lastKnownHealth      = 100
local G_autoPlayOnCountdown  = false
local SPAWN_Z_RIGHT_THRESHOLD = 60
local BR_L2 = Vector3.new(-475.5,  -3.75, 100.5)
local BR_L3 = Vector3.new(-486.5,  -3.75, 100.5)
local BR_R2 = Vector3.new(-475.50, -3.95, 17.55)
local BR_R3 = Vector3.new(-486.76, -3.95, 17.55)
local COUNTDOWN_TARGET = 4.9
autoStealEnabled=true   isStealing=false  stealStartTime=nil
autoStealConn=nil  progressConn=nil  STEAL_RADIUS=20  STEAL_DURATION=0.25
antiRagdollEnabled=true
lagModeEnabled=false  lagModeConn=nil
local LAGGER_SPEED=15
autoBatToggled=false
BAT_AIMBOT_SPEED=61  BAT_ENGAGE_RANGE=2.5  BAT_MELEE_OFFSET=1
_batAimbotConn=nil  _batLockedTarget=nil
local _batEquipLoopThread=nil
slowDownEnabled=false
tauntActive=false  tauntLoop=nil
infJumpEnabled=true  INF_JUMP_FORCE=54  CLAMP_FALL=80
gChar=nil  gHum=nil  gHrp=nil
ProgressBarFill=nil  ProgressLabel=nil  ProgressPctLabel=nil  RadiusInput=nil

toggleStates={}  mobBtnRefs={}
AntiRagdollConns={}
changingKeybind=nil
local dropBrainrotActive=false
local DROP_ASCEND_DURATION=0.2
local DROP_ASCEND_SPEED=150
local setDropBrainrotVisual=nil
local dropMobileSetter=nil
local floatEnabled=false
local UI_SCALE=0.5
local mobileButtonContainer=nil
local _sidePanelLocked=true

-- Desync speed variables (now uses working BodyVelocity desync)
local DS_NORMAL_SPEED = 999
local DS_CARRY_SPEED  = 999
local DS_savedNormal  = nil
local DS_savedCarry   = nil
local _dsLagbackConn  = nil

-- ── AUTOSAVE SYSTEM ──────────────────────────────────────────
local SAVE_FILE = "solo_settings.json"

local _btnRegistry = {}
local function _registerBtn(name, btn) _btnRegistry[name] = btn end

local function saveSettings()
    local data = {
        NORMAL_SPEED   = NORMAL_SPEED,
        SLOW_SPEED     = SLOW_SPEED,
        DS_NORMAL_SPEED= DS_NORMAL_SPEED,
        DS_CARRY_SPEED = DS_CARRY_SPEED,
        STEAL_RADIUS   = STEAL_RADIUS,
        STEAL_DURATION = STEAL_DURATION,
        UI_SCALE       = UI_SCALE,
        infJumpEnabled = infJumpEnabled,
        btnPositions   = {},
    }
    for name, btn in pairs(_btnRegistry) do
        if btn and btn.Parent then
            local p = btn.Position
            data.btnPositions[name] = {xs=p.X.Scale, xo=p.X.Offset, ys=p.Y.Scale, yo=p.Y.Offset}
        end
    end
    pcall(function()
        local encoded = game:GetService("HttpService"):JSONEncode(data)
        writefile(SAVE_FILE, encoded)
    end)
end

local function loadSettings()
    pcall(function()
        if isfile and isfile(SAVE_FILE) then
            local raw = readfile(SAVE_FILE)
            local data = game:GetService("HttpService"):JSONDecode(raw)
            if data.NORMAL_SPEED    then NORMAL_SPEED    = data.NORMAL_SPEED    end
            if data.SLOW_SPEED      then SLOW_SPEED      = data.SLOW_SPEED      end
            if data.DS_NORMAL_SPEED then DS_NORMAL_SPEED = data.DS_NORMAL_SPEED end
            if data.DS_CARRY_SPEED  then DS_CARRY_SPEED  = data.DS_CARRY_SPEED  end
            if data.STEAL_RADIUS    then STEAL_RADIUS    = data.STEAL_RADIUS    end
            if data.STEAL_DURATION  then STEAL_DURATION  = data.STEAL_DURATION  end
            if data.UI_SCALE        then UI_SCALE        = data.UI_SCALE        end
            if data.btnPositions then
                task.delay(0.5, function()
                    for name, pos in pairs(data.btnPositions) do
                        local btn = _btnRegistry[name]
                        if btn and btn.Parent then
                            btn.Position = UDim2.new(pos.xs, pos.xo, pos.ys, pos.yo)
                        end
                    end
                end)
            end
        end
    end)
end
loadSettings()

task.spawn(function()
    while task.wait(5) do saveSettings() end
end)

Keybinds={
    AutoLeft      = Enum.KeyCode.Q,
    AutoRight     = Enum.KeyCode.E,
    AutoSteal     = Enum.KeyCode.V,
    BatAimbot     = Enum.KeyCode.Z,
    AntiRagdoll   = Enum.KeyCode.X,
    LagMode       = Enum.KeyCode.N,
    SlowDown      = Enum.KeyCode.F7,
    Drop          = Enum.KeyCode.F3,
    Taunt         = Enum.KeyCode.F4,
    TPDown        = Enum.KeyCode.G,
    AutoPlayTimer = Enum.KeyCode.F6,
}
ControllerBinds={
    AutoLeft  = "X / □",
    AutoRight = "Y / △",
    BatAimbot = "B / ○",
    LagMode   = "L1",
    SlowDown  = "R1",
    TPDown    = "L2",
    Drop      = "R2",
    AutoSteal = "D←",
    AntiRagdoll="D→",
}
KeybindButtons={}

local function getHRP() local c=player.Character return c and c:FindFirstChild("HumanoidRootPart") end
local function getHum() local c=player.Character return c and c:FindFirstChildOfClass("Humanoid") end

local function playTick()
    local ts=Instance.new("Sound")
    ts.SoundId="rbxassetid://9119734881" ts.Volume=0.12
    ts.RollOffMaxDistance=0 ts.Parent=game:GetService("SoundService")
    ts:Play() game:GetService("Debris"):AddItem(ts,1)
end

-- ── INF JUMP ─────────────────────────────────────────────────
UserInputService.JumpRequest:Connect(function()
    if not infJumpEnabled then return end
    local h=getHRP() if not h then return end
    h.AssemblyLinearVelocity=Vector3.new(h.AssemblyLinearVelocity.X,INF_JUMP_FORCE,h.AssemblyLinearVelocity.Z)
end)
RunService.Heartbeat:Connect(function()
    if not infJumpEnabled then return end
    local h=getHRP() if not h then return end
    if h.AssemblyLinearVelocity.Y<-CLAMP_FALL then
        h.AssemblyLinearVelocity=Vector3.new(h.AssemblyLinearVelocity.X,-CLAMP_FALL,h.AssemblyLinearVelocity.Z)
    end
end)

-- ── TP DOWN (J5 raycast version) ─────────────────────────────
local setTPDownVisual=nil
local function runTPDown()
    task.spawn(function()
        local c=player.Character
        local hrp=c and c:FindFirstChild("HumanoidRootPart")
        local hum=c and c:FindFirstChildOfClass("Humanoid")
        if not c or not hrp or not hum then
            if setTPDownVisual then setTPDownVisual(false) end
            return
        end
        local origin=hrp.Position+Vector3.new(0,2,0)
        local rp=RaycastParams.new()
        rp.FilterDescendantsInstances={c}
        rp.FilterType=Enum.RaycastFilterType.Exclude
        local hit=workspace:Raycast(origin,Vector3.new(0,-1000,0),rp)
        if hit then
            local hh=hum.HipHeight or 2
            local hy=hrp.Size.Y/2
            local targetCF=CFrame.new(hrp.Position.X,hit.Position.Y+hh+hy+0.1,hrp.Position.Z)
            pcall(function() hrp.AssemblyLinearVelocity=Vector3.zero end)
            pcall(function() hrp.AssemblyAngularVelocity=Vector3.zero end)
            pcall(function()
                c:PivotTo(targetCF*(c:GetPivot():Inverse()*hrp.CFrame):Inverse()*c:GetPivot())
            end)
            pcall(function() hrp.CFrame=targetCF end)
            for _=1,6 do
                task.wait()
                pcall(function()
                    hrp.AssemblyLinearVelocity=Vector3.zero
                    hrp.AssemblyAngularVelocity=Vector3.zero
                end)
            end
        end
        if setTPDownVisual then setTPDownVisual(false) end
    end)
end

-- ── DROP BRAINROT ─────────────────────────────────────────────
local function runDropBrainrot()
    if dropBrainrotActive then return end
    local char=player.Character if not char then return end
    local root=char:FindFirstChild("HumanoidRootPart") if not root then return end
    dropBrainrotActive=true
    local t0=tick()
    local dc
    dc=RunService.Heartbeat:Connect(function()
        local r=char and char:FindFirstChild("HumanoidRootPart")
        if not r then
            dc:Disconnect() dropBrainrotActive=false
            if setDropBrainrotVisual then setDropBrainrotVisual(false) end
            if dropMobileSetter then dropMobileSetter(false) end
            return
        end
        if tick()-t0>=DROP_ASCEND_DURATION then
            dc:Disconnect()
            local rp=RaycastParams.new()
            rp.FilterDescendantsInstances={char}
            rp.FilterType=Enum.RaycastFilterType.Exclude
            local rr=workspace:Raycast(r.Position,Vector3.new(0,-2000,0),rp)
            if rr then
                local hum2=char:FindFirstChildOfClass("Humanoid")
                local off=(hum2 and hum2.HipHeight or 2)+(r.Size.Y/2)
                r.CFrame=CFrame.new(r.Position.X,rr.Position.Y+off,r.Position.Z)
                r.AssemblyLinearVelocity=Vector3.new(0,0,0)
            end
            dropBrainrotActive=false
            if setDropBrainrotVisual then setDropBrainrotVisual(false) end
            if dropMobileSetter then dropMobileSetter(false) end
            return
        end
        r.AssemblyLinearVelocity=Vector3.new(r.AssemblyLinearVelocity.X,DROP_ASCEND_SPEED,r.AssemblyLinearVelocity.Z)
    end)
end

-- ── TAUNT ─────────────────────────────────────────────────────
local function fireChat(msg)
    local ok=false
    pcall(function()
        local TCS=game:GetService("TextChatService")
        local ch=TCS.TextChannels:FindFirstChild("RBXGeneral") or TCS.TextChannels:FindFirstChildWhichIsA("TextChannel")
        if ch then ch:SendAsync(msg) ok=true end
    end)
    if not ok then pcall(function() game:GetService("ReplicatedStorage").DefaultChatSystemChatEvents.SayMessageRequest:FireServer(msg,"All") end) end
end
local function startTaunt()
    if tauntLoop then return end tauntActive=true
    tauntLoop=task.spawn(function()
        while tauntActive do fireChat("ZERO ON TOP") task.wait(0.5) end
    end)
end
local function stopTaunt()
    tauntActive=false if tauntLoop then task.cancel(tauntLoop) tauntLoop=nil end
end

-- ── ANTI RAGDOLL ──────────────────────────────────────────────
local ANTI_FLING_MAX_XZ = 80
local ANTI_FLING_MAX_Y  = 120
local _arConn = nil

local function startAntiRagdoll()
    if _arConn then return end
    _arConn = RunService.Heartbeat:Connect(function()
        if not antiRagdollEnabled then return end
        if autoBatToggled then return end
        local c    = player.Character;                  if not c   then return end
        local root = c:FindFirstChild("HumanoidRootPart")
        local hum  = c:FindFirstChildOfClass("Humanoid"); if not hum then return end
        local st   = hum:GetState()
        local isRag = st==Enum.HumanoidStateType.Physics
                   or st==Enum.HumanoidStateType.Ragdoll
                   or st==Enum.HumanoidStateType.FallingDown
                   or st==Enum.HumanoidStateType.GettingUp
        if isRag then
            hum:ChangeState(Enum.HumanoidStateType.Running)
            workspace.CurrentCamera.CameraSubject = hum
            pcall(function()
                local pm = player.PlayerScripts:FindFirstChild("PlayerModule")
                if pm then require(pm):GetControls():Enable() end
            end)
            if root then
                root.AssemblyLinearVelocity  = Vector3.new(0,0,0)
                root.AssemblyAngularVelocity = Vector3.new(0,0,0)
            end
        end
        for _,obj in ipairs(c:GetDescendants()) do
            if obj:IsA("Motor6D") and not obj.Enabled then obj.Enabled=true end
        end
        if root then
            local vel   = root.AssemblyLinearVelocity
            local xzMag = Vector3.new(vel.X,0,vel.Z).Magnitude
            local cx,cz = vel.X,vel.Z
            if xzMag > ANTI_FLING_MAX_XZ then
                local sc=ANTI_FLING_MAX_XZ/xzMag; cx=vel.X*sc; cz=vel.Z*sc
            end
            local cy = math.clamp(vel.Y,-ANTI_FLING_MAX_Y,ANTI_FLING_MAX_Y)
            if math.abs(cx-vel.X)>0.5 or math.abs(cz-vel.Z)>0.5 or math.abs(cy-vel.Y)>0.5 then
                root.AssemblyLinearVelocity  = Vector3.new(cx,cy,cz)
                root.AssemblyAngularVelocity = Vector3.new(0,0,0)
            end
        end
    end)
end

local function stopAntiRagdoll()
    if _arConn then _arConn:Disconnect(); _arConn=nil end
    AntiRagdollConns={}
end

-- ── LAG MODE ──────────────────────────────────────────────────
local _lagModeDescConn = nil
local _lagSavedAnimate = nil

local function startLagMode()
    lagModeEnabled = true
    local c = player.Character
    if c then
        local hum = c:FindFirstChildOfClass("Humanoid")
        if hum then
            for _, t in ipairs(hum:GetPlayingAnimationTracks()) do pcall(function() t:Stop(0) end) end
        end
        local anim = c:FindFirstChild("Animate")
        if anim then
            _lagSavedAnimate = anim:Clone()
            anim:Destroy()
        end
    end
    if lagModeConn then lagModeConn:Disconnect() lagModeConn = nil end
    lagModeConn = RunService.Heartbeat:Connect(function()
        if not lagModeEnabled then
            if lagModeConn then lagModeConn:Disconnect() lagModeConn = nil end
            return
        end
        local cc  = player.Character if not cc then return end
        local hh  = cc:FindFirstChildOfClass("Humanoid") if not hh then return end
        local hrp = cc:FindFirstChild("HumanoidRootPart") if not hrp then return end
        if hh.WalkSpeed ~= LAGGER_SPEED then hh.WalkSpeed = LAGGER_SPEED end
        local md = hh.MoveDirection
        if md.Magnitude > 0.01 then
            hrp.AssemblyLinearVelocity = Vector3.new(
                md.X * LAGGER_SPEED,
                hrp.AssemblyLinearVelocity.Y,
                md.Z * LAGGER_SPEED
            )
        end
    end)
    if _lagModeDescConn then _lagModeDescConn:Disconnect() _lagModeDescConn = nil end
    _lagModeDescConn = player.CharacterAdded:Connect(function(newChar)
        if not lagModeEnabled then return end
        task.wait(0.4)
        local hum2 = newChar:FindFirstChildOfClass("Humanoid")
        if hum2 then
            for _, t in ipairs(hum2:GetPlayingAnimationTracks()) do pcall(function() t:Stop(0) end) end
            hum2.WalkSpeed = LAGGER_SPEED
        end
        local anim2 = newChar:FindFirstChild("Animate")
        if anim2 then _lagSavedAnimate = anim2:Clone(); anim2:Destroy() end
    end)
end

local function stopLagMode()
    lagModeEnabled = false
    if lagModeConn then lagModeConn:Disconnect() lagModeConn = nil end
    if _lagModeDescConn then _lagModeDescConn:Disconnect() _lagModeDescConn = nil end
    local c = player.Character
    if c and _lagSavedAnimate then
        local existing = c:FindFirstChild("Animate")
        if existing then existing:Destroy() end
        _lagSavedAnimate:Clone().Parent = c
    end
    _lagSavedAnimate = nil
    local hum = c and c:FindFirstChildOfClass("Humanoid")
    if hum then
        if desyncActive then
            hum.WalkSpeed = slowDownEnabled and DS_CARRY_SPEED or DS_NORMAL_SPEED
        else
            hum.WalkSpeed = slowDownEnabled and SLOW_SPEED or NORMAL_SPEED
        end
    end
end

-- ── BAT AIMBOT (Demon Hub Engine) ─────────────────────────────
local _demonAimbotLockedTarget = nil
local _demonAimbotHighlight = Instance.new("Highlight")
_demonAimbotHighlight.Name = "DemonAimbotESP"
_demonAimbotHighlight.FillColor = Color3.fromRGB(0,180,0)
_demonAimbotHighlight.OutlineColor = Color3.fromRGB(80,255,80)
_demonAimbotHighlight.FillTransparency = 0.5
_demonAimbotHighlight.OutlineTransparency = 0
pcall(function() _demonAimbotHighlight.Parent = player:WaitForChild("PlayerGui") end)

local DEMON_MELEE_OFFSET    = 2.2
local DEMON_SWING_COOLDOWN  = 0.08
local _demonLastSwing       = 0
local _demonLastEquip       = 0

local function _demonFindBat()
    local c  = player.Character
    local bp = player:FindFirstChildOfClass("Backpack")
    local SlapList = {"Bat","Slap","Iron Slap","Gold Slap","Diamond Slap","Emerald Slap","Ruby Slap","Dark Matter Slap","Flame Slap","Nuclear Slap","Galaxy Slap","Glitched Slap"}
    if c then for _,ch in ipairs(c:GetChildren()) do if ch:IsA("Tool") and ch.Name:lower():find("bat") then return ch end end end
    if bp then for _,ch in ipairs(bp:GetChildren()) do if ch:IsA("Tool") and ch.Name:lower():find("bat") then return ch end end end
    for _,name in ipairs(SlapList) do
        local t = (c and c:FindFirstChild(name)) or (bp and bp:FindFirstChild(name))
        if t then return t end
    end
end

local function _demonIsTargetValid(tc)
    if not tc or not tc.Parent then return false end
    local hum = tc:FindFirstChildOfClass("Humanoid")
    local hrp = tc:FindFirstChild("HumanoidRootPart")
    local ff  = tc:FindFirstChildOfClass("ForceField")
    return hum and hrp and hum.Health > 0 and not ff
end

local function _demonGetBestTarget(myHRP)
    local best, bestHRP, bestDist = nil, nil, math.huge
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= player and _demonIsTargetValid(p.Character) then
            local eh = p.Character:FindFirstChild("HumanoidRootPart")
            if eh then
                local d = (eh.Position - myHRP.Position).Magnitude
                if d < bestDist then bestDist = d; bestHRP = eh; best = p.Character end
            end
        end
    end
    if best ~= _demonAimbotLockedTarget then
        local switchToNew = true
        if _demonAimbotLockedTarget and _demonIsTargetValid(_demonAimbotLockedTarget) then
            local curHRP = _demonAimbotLockedTarget:FindFirstChild("HumanoidRootPart")
            if curHRP then
                local curDist = (curHRP.Position - myHRP.Position).Magnitude
                if bestDist >= curDist - 3 then switchToNew = false end
            end
        end
        if switchToNew then
            _demonAimbotLockedTarget = best
            local newBat = _demonFindBat()
            local cc = player.Character
            local hm = cc and cc:FindFirstChildOfClass("Humanoid")
            if newBat and hm and newBat.Parent ~= cc then
                pcall(function() hm:EquipTool(newBat) end)
            end
        end
    end
    local lockedHRP = _demonAimbotLockedTarget and _demonAimbotLockedTarget:FindFirstChild("HumanoidRootPart")
    return lockedHRP, _demonAimbotLockedTarget
end

local function startBatAimbot()
    if _batAimbotConn then _batAimbotConn:Disconnect() _batAimbotConn = nil end
    local c   = player.Character if not c then return end
    local h   = c:FindFirstChild("HumanoidRootPart")
    local hum = c:FindFirstChildOfClass("Humanoid")
    if not h or not hum then return end

    hum.AutoRotate = false
    autoBatToggled = true
    G_lastKnownHealth = hum.Health
    _demonAimbotLockedTarget = nil

    for _, name in ipairs({"AimbotAttachment","AimbotAlign","J5AimbotAtt","J5AimbotAln","J5AimbotLV","J5AimbotLA"}) do
        local old = h:FindFirstChild(name) if old then old:Destroy() end
    end

    local att = Instance.new("Attachment", h)
    att.Name = "AimbotAttachment"
    local align = Instance.new("AlignOrientation", h)
    align.Name           = "AimbotAlign"
    align.Mode           = Enum.OrientationAlignmentMode.OneAttachment
    align.Attachment0    = att
    align.MaxTorque      = 40000
    align.Responsiveness = 60

    local _wasGrounded = true

    _batAimbotConn = RunService.Heartbeat:Connect(function(dt)
        if not autoBatToggled then return end
        local cc  = player.Character if not cc then return end
        local hrp = cc:FindFirstChild("HumanoidRootPart") if not hrp then return end
        local hm2 = cc:FindFirstChildOfClass("Humanoid")  if not hm2 then return end

        local bat = _demonFindBat()
        if bat and bat.Parent ~= cc then
            local now2 = tick()
            if now2 - _demonLastEquip >= 0.5 then
                _demonLastEquip = now2
                pcall(function() hm2:EquipTool(bat) end)
            end
        end

        local now = tick()
        local targetHRP, targetChar = _demonGetBestTarget(hrp)

        if targetHRP and targetChar then
            _demonAimbotHighlight.Adornee = targetChar

            local vel   = targetHRP.AssemblyLinearVelocity
            local spd   = vel.Magnitude
            local toMe  = (hrp.Position - targetHRP.Position)
            local dist3D = toMe.Magnitude
            local predictedPos
            if spd > 0.1 and toMe.Magnitude > 0.01 then
                local crossFactor = 1 - math.abs(vel.Unit:Dot(toMe.Unit))
                local tPred = math.clamp((spd / 60) * (0.6 + crossFactor * 0.6), 0.04, 0.28)
                predictedPos = targetHRP.Position + vel * tPred
            else
                predictedPos = targetHRP.Position
            end

            local lookDir = (predictedPos - hrp.Position)
            if lookDir.Magnitude > 0.01 then
                align.CFrame = CFrame.lookAt(hrp.Position, hrp.Position + lookDir)
            end
            hm2.AutoRotate = false

            local toTarget = predictedPos - hrp.Position
            local standPos = dist3D > DEMON_MELEE_OFFSET
                and (predictedPos - toTarget.Unit * DEMON_MELEE_OFFSET)
                or predictedPos

            local moveDir  = standPos - hrp.Position
            local moveDist = moveDir.Magnitude

            local myVelY = hrp.AssemblyLinearVelocity.Y
            _wasGrounded = math.abs(myVelY) < 4

            if moveDist > 0.4 then
                local adaptiveSpd = BAT_AIMBOT_SPEED * math.clamp(moveDist / 8, 0.5, 1.0)
                local dir = moveDir.Unit
                local yComponent
                if _wasGrounded and math.abs(dir.Y) < 0.35 then
                    yComponent = myVelY
                else
                    yComponent = dir.Y * adaptiveSpd
                end
                hrp.AssemblyLinearVelocity = Vector3.new(
                    dir.X * adaptiveSpd,
                    yComponent,
                    dir.Z * adaptiveSpd
                )
            else
                hrp.AssemblyLinearVelocity = Vector3.new(vel.X, _wasGrounded and myVelY or vel.Y, vel.Z)
            end

            bat = _demonFindBat()
            if bat and bat.Parent == cc and dist3D <= DEMON_MELEE_OFFSET + 2.5 then
                if now - _demonLastSwing >= DEMON_SWING_COOLDOWN then
                    _demonLastSwing = now
                    pcall(function() bat:Activate() end)
                end
            end
        else
            _demonAimbotLockedTarget = nil
            _demonAimbotHighlight.Adornee = nil
        end
    end)
end

local function stopBatAimbot()
    autoBatToggled = false

    if _batEquipLoopThread then
        pcall(function() task.cancel(_batEquipLoopThread) end)
        _batEquipLoopThread = nil
    end

    if _batAimbotConn then _batAimbotConn:Disconnect() _batAimbotConn = nil end

    local c   = player.Character
    local h   = c and c:FindFirstChild("HumanoidRootPart")
    local hum = c and c:FindFirstChildOfClass("Humanoid")

    if h then
        for _, name in ipairs({"AimbotAttachment","AimbotAlign","J5AimbotAtt","J5AimbotAln","J5AimbotLV","J5AimbotLA"}) do
            local obj = h:FindFirstChild(name) if obj then obj:Destroy() end
        end
        pcall(function()
            h.AssemblyLinearVelocity  = Vector3.new(0, h.AssemblyLinearVelocity.Y, 0)
            h.AssemblyAngularVelocity = Vector3.zero
        end)
    end

    if hum then
        hum.PlatformStand = false
        hum.AutoRotate    = true
        hum:Move(Vector3.zero, false)
        G_lastKnownHealth = hum.Health
    end

    _demonAimbotLockedTarget   = nil
    _demonAimbotHighlight.Adornee = nil

    task.spawn(function()
        for _ = 1, 3 do
            task.wait()
            local cc  = player.Character if not cc then break end
            local hm2 = cc:FindFirstChildOfClass("Humanoid")
            local hrp2= cc:FindFirstChild("HumanoidRootPart")
            if hm2 then hm2.PlatformStand = false; hm2.AutoRotate = true end
            if hrp2 then
                pcall(function()
                    hrp2.AssemblyLinearVelocity  = Vector3.new(0, hrp2.AssemblyLinearVelocity.Y, 0)
                    hrp2.AssemblyAngularVelocity = Vector3.zero
                end)
            end
        end
    end)
end

-- ── AUTO PLAY (Eclipse logic) ─────────────────────────────────
local function stopAutoPlayLeft()
    aplOn=false if aplConn then aplConn:Disconnect() aplConn=nil end aplPhase=1
    local hh=getHum() if hh then hh:Move(Vector3.zero,false) end
end
local function stopAutoPlayRight()
    aprOn=false if aprConn then aprConn:Disconnect() aprConn=nil end aprPhase=1
    local hh=getHum() if hh then hh:Move(Vector3.zero,false) end
end
local function startAutoPlayLeft()
    if aplConn then aplConn:Disconnect() end aplPhase=1
    local _aplPhase0Start=0
    aplConn=RunService.Heartbeat:Connect(function()
        if not aplOn or not gHrp or not gHum then return end
        if aplPhase==1 then
            local d=Vector3.new(POS_L1.X-gHrp.Position.X,0,POS_L1.Z-gHrp.Position.Z)
            if d.Magnitude<1 then aplPhase=2 return end
            local md=d.Unit gHum:Move(md,false)
            gHrp.AssemblyLinearVelocity=Vector3.new(md.X*NORMAL_SPEED,gHrp.AssemblyLinearVelocity.Y,md.Z*NORMAL_SPEED)
        elseif aplPhase==2 then
            local d=Vector3.new(POS_L2.X-gHrp.Position.X,0,POS_L2.Z-gHrp.Position.Z)
            if d.Magnitude<1 then
                aplPhase=0 _aplPhase0Start=tick()
                gHum:Move(Vector3.zero,false) gHrp.AssemblyLinearVelocity=Vector3.zero
                return
            end
            local md=d.Unit gHum:Move(md,false)
            gHrp.AssemblyLinearVelocity=Vector3.new(md.X*NORMAL_SPEED,gHrp.AssemblyLinearVelocity.Y,md.Z*NORMAL_SPEED)
        elseif aplPhase==0 then
            if tick()-_aplPhase0Start>=0.05 then aplPhase=3 end
            return
        elseif aplPhase==3 then
            local d=Vector3.new(EXIT_L.X-gHrp.Position.X,0,EXIT_L.Z-gHrp.Position.Z)
            if d.Magnitude<1 then aplPhase=4 return end
            local md=d.Unit gHum:Move(md,false)
            gHrp.AssemblyLinearVelocity=Vector3.new(md.X*SLOW_SPEED,gHrp.AssemblyLinearVelocity.Y,md.Z*SLOW_SPEED)
        elseif aplPhase==4 then
            local d=Vector3.new(LFINAL.X-gHrp.Position.X,0,LFINAL.Z-gHrp.Position.Z)
            if d.Magnitude<1 then
                gHum:Move(Vector3.zero,false) gHrp.AssemblyLinearVelocity=Vector3.zero
                aplOn=false aplPhase=1
                if aplConn then aplConn:Disconnect() aplConn=nil end
                if VisualSetters and VisualSetters.AutoLeft then VisualSetters.AutoLeft(false) end
                if mobBtnRefs and mobBtnRefs["AutoLeft"] then
                    TweenService:Create(mobBtnRefs["AutoLeft"],TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(0,0,0)}):Play()
                end
                return
            end
            local md=d.Unit gHum:Move(md,false)
            gHrp.AssemblyLinearVelocity=Vector3.new(md.X*SLOW_SPEED,gHrp.AssemblyLinearVelocity.Y,md.Z*SLOW_SPEED)
        end
    end)
end
local function startAutoPlayRight()
    if aprConn then aprConn:Disconnect() end aprPhase=1
    local _aprPhase0Start=0
    aprConn=RunService.Heartbeat:Connect(function()
        if not aprOn or not gHrp or not gHum then return end
        if aprPhase==1 then
            local d=Vector3.new(POS_R1.X-gHrp.Position.X,0,POS_R1.Z-gHrp.Position.Z)
            if d.Magnitude<1 then aprPhase=2 return end
            local md=d.Unit gHum:Move(md,false)
            gHrp.AssemblyLinearVelocity=Vector3.new(md.X*NORMAL_SPEED,gHrp.AssemblyLinearVelocity.Y,md.Z*NORMAL_SPEED)
        elseif aprPhase==2 then
            local d=Vector3.new(POS_R2.X-gHrp.Position.X,0,POS_R2.Z-gHrp.Position.Z)
            if d.Magnitude<1 then
                aprPhase=0 _aprPhase0Start=tick()
                gHum:Move(Vector3.zero,false) gHrp.AssemblyLinearVelocity=Vector3.zero
                return
            end
            local md=d.Unit gHum:Move(md,false)
            gHrp.AssemblyLinearVelocity=Vector3.new(md.X*NORMAL_SPEED,gHrp.AssemblyLinearVelocity.Y,md.Z*NORMAL_SPEED)
        elseif aprPhase==0 then
            if tick()-_aprPhase0Start>=0.05 then aprPhase=3 end
            return
        elseif aprPhase==3 then
            local d=Vector3.new(EXIT_R.X-gHrp.Position.X,0,EXIT_R.Z-gHrp.Position.Z)
            if d.Magnitude<1 then aprPhase=4 return end
            local md=d.Unit gHum:Move(md,false)
            gHrp.AssemblyLinearVelocity=Vector3.new(md.X*SLOW_SPEED,gHrp.AssemblyLinearVelocity.Y,md.Z*SLOW_SPEED)
        elseif aprPhase==4 then
            local d=Vector3.new(RFINAL.X-gHrp.Position.X,0,RFINAL.Z-gHrp.Position.Z)
            if d.Magnitude<1 then
                gHum:Move(Vector3.zero,false) gHrp.AssemblyLinearVelocity=Vector3.zero
                aprOn=false aprPhase=1
                if aprConn then aprConn:Disconnect() aprConn=nil end
                if VisualSetters and VisualSetters.AutoRight then VisualSetters.AutoRight(false) end
                if mobBtnRefs and mobBtnRefs["AutoRight"] then
                    TweenService:Create(mobBtnRefs["AutoRight"],TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(0,0,0)}):Play()
                end
                return
            end
            local md=d.Unit gHum:Move(md,false)
            gHrp.AssemblyLinearVelocity=Vector3.new(md.X*SLOW_SPEED,gHrp.AssemblyLinearVelocity.Y,md.Z*SLOW_SPEED)
        end
    end)
end

-- ── AUTO STEAL ─────────────────────────────────────────────────
local _stealData       = {}
local _lastStealTick   = 0
local _plotCache       = {}
local _plotCacheTime   = {}
local _cachedPrompts   = {}
local _promptCacheTime = 0
local STEAL_COOLDOWN        = 0.1
local PLOT_CACHE_DURATION   = 2
local PROMPT_CACHE_REFRESH  = 0.15

local function _isMyPlotByName(plotName)
    local now = tick()
    if _plotCache[plotName] and (now - (_plotCacheTime[plotName] or 0)) < PLOT_CACHE_DURATION then
        return _plotCache[plotName]
    end
    local plots = workspace:FindFirstChild("Plots")
    if not plots then _plotCache[plotName]=false _plotCacheTime[plotName]=now return false end
    local plot = plots:FindFirstChild(plotName)
    if not plot then _plotCache[plotName]=false _plotCacheTime[plotName]=now return false end
    local sign = plot:FindFirstChild("PlotSign")
    if sign then
        local yb = sign:FindFirstChild("YourBase")
        if yb and yb:IsA("BillboardGui") then
            local result = yb.Enabled == true
            _plotCache[plotName]=result _plotCacheTime[plotName]=now
            return result
        end
    end
    _plotCache[plotName]=false _plotCacheTime[plotName]=now return false
end

local function _findNearestPrompt()
    local root = getHRP() if not root then return nil end
    local now = tick()
    if now - _promptCacheTime < PROMPT_CACHE_REFRESH and #_cachedPrompts > 0 then
        local best, bestDist, bestName = nil, math.huge, nil
        for _, data in ipairs(_cachedPrompts) do
            if data.spawn then
                local d = (data.spawn.Position - root.Position).Magnitude
                if d <= STEAL_RADIUS and d < bestDist then
                    best=data.prompt bestDist=d bestName=data.name
                end
            end
        end
        if best then return best, bestDist, bestName end
    end
    _cachedPrompts={} _promptCacheTime=now
    local plots = workspace:FindFirstChild("Plots") if not plots then return nil end
    local best, bestDist, bestName = nil, math.huge, nil
    for _, plot in ipairs(plots:GetChildren()) do
        if _isMyPlotByName(plot.Name) then continue end
        local podiums = plot:FindFirstChild("AnimalPodiums") if not podiums then continue end
        for _, podium in ipairs(podiums:GetChildren()) do
            pcall(function()
                local base  = podium:FindFirstChild("Base")
                local spawn = base and base:FindFirstChild("Spawn")
                if spawn then
                    local d = (spawn.Position - root.Position).Magnitude
                    local att = spawn:FindFirstChild("PromptAttachment")
                    if att then
                        for _, ch in ipairs(att:GetChildren()) do
                            if ch:IsA("ProximityPrompt") then
                                table.insert(_cachedPrompts,{prompt=ch,spawn=spawn,name=podium.Name})
                                if d <= STEAL_RADIUS and d < bestDist then
                                    best=ch bestDist=d bestName=podium.Name
                                end
                                break
                            end
                        end
                    end
                end
            end)
        end
    end
    return best, bestDist, bestName
end

local function _executeSteal(prompt, name)
    local now = tick()
    if now - _lastStealTick < STEAL_COOLDOWN then return end
    if isStealing then return end
    if not _stealData[prompt] then
        _stealData[prompt] = {hold={}, trigger={}, ready=true}
        pcall(function()
            if getconnections then
                for _, c in ipairs(getconnections(prompt.PromptButtonHoldBegan)) do
                    if c.Function then table.insert(_stealData[prompt].hold, c.Function) end
                end
                for _, c in ipairs(getconnections(prompt.Triggered)) do
                    if c.Function then table.insert(_stealData[prompt].trigger, c.Function) end
                end
            else _stealData[prompt].useFallback = true end
        end)
    end
    local data = _stealData[prompt]
    if not data.ready then return end
    data.ready=false isStealing=true stealStartTime=now _lastStealTick=now
    if ProgressLabel then ProgressLabel.Text = name or "STEALING..." end
    if progressConn then progressConn:Disconnect() end
    progressConn = RunService.Heartbeat:Connect(function()
        if not isStealing then progressConn:Disconnect() return end
        local prog = math.clamp((tick()-stealStartTime)/STEAL_DURATION, 0, 1)
        if ProgressBarFill  then ProgressBarFill.Size = UDim2.new(prog,0,1,0) end
        if ProgressPctLabel then ProgressPctLabel.Text = math.floor(prog*100).."%" end
    end)
    task.spawn(function()
        local ok = false
        pcall(function()
            if not data.useFallback then
                for _, f in ipairs(data.hold) do task.spawn(f) end
                task.wait(STEAL_DURATION)
                for _, f in ipairs(data.trigger) do task.spawn(f) end
                ok = true
            end
        end)
        if not ok and fireproximityprompt then
            pcall(function() fireproximityprompt(prompt) ok=true end)
        end
        if not ok then
            pcall(function()
                prompt:InputHoldBegin() task.wait(STEAL_DURATION) prompt:InputHoldEnd() ok=true
            end)
        end
        task.wait(STEAL_DURATION * 0.3)
        if progressConn then progressConn:Disconnect() end
        if ProgressLabel    then ProgressLabel.Text = "READY" end
        if ProgressPctLabel then ProgressPctLabel.Text = "" end
        if ProgressBarFill  then ProgressBarFill.Size = UDim2.new(0,0,1,0) end
        task.wait(0.05)
        data.ready=true isStealing=false
    end)
end

local function startAutoSteal()
    if autoStealConn then return end
    autoStealConn = RunService.Heartbeat:Connect(function()
        if not autoStealEnabled or isStealing then return end
        local p, _, n = _findNearestPrompt()
        if p then
            local hrp = getHRP()
            if hrp then
                hrp.AssemblyLinearVelocity = Vector3.new(0, hrp.AssemblyLinearVelocity.Y, 0)
            end
            _executeSteal(p, n)
        end
    end)
end
local function stopAutoSteal()
    if autoStealConn then autoStealConn:Disconnect() autoStealConn=nil end
    isStealing=false _lastStealTick=0
    _plotCache={} _plotCacheTime={} _cachedPrompts={}
    if progressConn then progressConn:Disconnect() progressConn=nil end
    if ProgressBarFill  then ProgressBarFill.Size = UDim2.new(0,0,1,0) end
    if ProgressLabel    then ProgressLabel.Text = "READY" end
    if ProgressPctLabel then ProgressPctLabel.Text = "" end
end

-- ── AUTO TP SYSTEM ────────────────────────────────────────────
local function detectMyPlot()
    local plots=workspace:FindFirstChild("Plots") if not plots then return nil,nil end
    local myName=player.DisplayName or player.Name
    for _,plot in ipairs(plots:GetChildren()) do
        local ok,result=pcall(function()
            local sign=plot:FindFirstChild("PlotSign") if not sign then return nil end
            local sg=sign:FindFirstChild("SurfaceGui") if not sg then return nil end
            local fr=sg:FindFirstChild("Frame") if not fr then return nil end
            local tl=fr:FindFirstChild("TextLabel") if not tl then return nil end
            if tl.Text:find(myName,1,true) then
                local spawnObj=plot:FindFirstChild("Spawn")
                if spawnObj then
                    local z=spawnObj.CFrame.Position.Z
                    return z<SPAWN_Z_RIGHT_THRESHOLD and "left" or "right"
                end
            end
            return nil
        end)
        if ok and result then return result,plot.Name end
    end
    return nil,nil
end

local function refreshMyPlotSide()
    local side,plotName=detectMyPlot()
    G_myPlotSide=side G_myPlotName=plotName
    return side
end

task.spawn(function()
    while true do
        task.wait(2)
        if G_tpAutoEnabled then refreshMyPlotSide() end
    end
end)

task.spawn(function()
    local plots=workspace:WaitForChild("Plots",30) if not plots then return end
    local myName=player.DisplayName or player.Name
    local function watchPlot(plot)
        pcall(function()
            local tl=plot:WaitForChild("PlotSign",5):WaitForChild("SurfaceGui",5):WaitForChild("Frame",5):WaitForChild("TextLabel",5)
            if not tl then return end
            tl:GetPropertyChangedSignal("Text"):Connect(function() refreshMyPlotSide() end)
            if tl.Text:find(myName,1,true) then refreshMyPlotSide() end
        end)
    end
    for _,plot in ipairs(plots:GetChildren()) do task.spawn(watchPlot,plot) end
    plots.ChildAdded:Connect(function(plot) task.spawn(watchPlot,plot) end)
end)

local function doReturnTeleport(side)
    if G_tpReturnCooldown then return end
    G_tpReturnCooldown=true
    task.spawn(function()
        pcall(function()
            local c=player.Character if not c then return end
            local root=c:FindFirstChild("HumanoidRootPart")
            local hum=c:FindFirstChildOfClass("Humanoid")
            if not root then return end
            local rotation=(side=="right") and math.rad(180) or 0
            local step2=(side=="right") and BR_R2 or BR_L2
            local step3=(side=="right") and BR_R3 or BR_L3
            local function tp(pos)
                root.AssemblyLinearVelocity=Vector3.zero
                root.AssemblyAngularVelocity=Vector3.zero
                root.CFrame=CFrame.new(pos+Vector3.new(0,3,0))*CFrame.Angles(0,rotation,0)
                if hum then hum:ChangeState(Enum.HumanoidStateType.Running) hum:Move(Vector3.zero,false) end
                for _,obj in ipairs(c:GetDescendants()) do
                    if obj:IsA("Motor6D") and not obj.Enabled then obj.Enabled=true end
                end
            end
            tp(step2) task.wait(0.1) tp(step3)
        end)
        local _c=player.Character
        local _hum=_c and _c:FindFirstChildOfClass("Humanoid")
        if _hum then G_lastKnownHealth=_hum.Health end
        G_tpReturnCooldown=false
        if G_autoPlayAfterTP and G_myPlotSide then
            task.wait(0.3)
            local s=G_myPlotSide
            if aplOn then stopAutoPlayLeft() aplOn=false if VisualSetters.AutoLeft then VisualSetters.AutoLeft(false) end end
            if aprOn then stopAutoPlayRight() aprOn=false if VisualSetters.AutoRight then VisualSetters.AutoRight(false) end end
            task.wait(0.1)
            if s=="left" then
                aplOn=true startAutoPlayLeft()
                if VisualSetters.AutoLeft then VisualSetters.AutoLeft(true) end
            elseif s=="right" then
                aprOn=true startAutoPlayRight()
                if VisualSetters.AutoRight then VisualSetters.AutoRight(true) end
            end
        end
    end)
end

RunService.Heartbeat:Connect(function()
    if not G_tpAutoEnabled then return end
    if G_tpReturnCooldown then return end
    if G_countdownActive then return end
    local c=player.Character if not c then return end
    local hum=c:FindFirstChildOfClass("Humanoid") if not hum then return end
    local hrp=c:FindFirstChild("HumanoidRootPart")
    if hrp and hrp.Anchored then G_lastKnownHealth=hum.Health return end
    if autoBatToggled then G_lastKnownHealth=hum.Health return end
    local hp=hum.Health
    local st=hum:GetState()
    local rag=st==Enum.HumanoidStateType.Physics or st==Enum.HumanoidStateType.Ragdoll or st==Enum.HumanoidStateType.FallingDown
    local wasHit=hp<G_lastKnownHealth-1 G_lastKnownHealth=hp
    if not (wasHit or rag) then return end
    if aplOn then stopAutoPlayLeft() aplOn=false if VisualSetters.AutoLeft then VisualSetters.AutoLeft(false) end end
    if aprOn then stopAutoPlayRight() aprOn=false if VisualSetters.AutoRight then VisualSetters.AutoRight(false) end end
    local s=G_myPlotSide
    if s=="left" then doReturnTeleport("left")
    elseif s=="right" then doReturnTeleport("right") end
end)

local function monitorCountdown(snd)
    if snd.Name~="Countdown" or not snd:IsA("Sound") then return end
    local triggered,conn=false,nil
    G_countdownActive=true
    conn=RunService.Heartbeat:Connect(function()
        if not snd or not snd.Parent or not snd.Playing then
            G_countdownActive=false
            if conn then conn:Disconnect() conn=nil end return
        end
        local ct=snd.TimePosition
        if ct>=COUNTDOWN_TARGET and not triggered then
            triggered=true G_countdownActive=false
            if conn then conn:Disconnect() conn=nil end
            if not G_autoPlayOnCountdown then return end
            local s=G_myPlotSide if not s then return end
            if aplOn then stopAutoPlayLeft() aplOn=false if VisualSetters.AutoLeft then VisualSetters.AutoLeft(false) end end
            if aprOn then stopAutoPlayRight() aprOn=false if VisualSetters.AutoRight then VisualSetters.AutoRight(false) end end
            task.wait(0.1)
            if s=="left" then
                aprOn=true startAutoPlayRight()
                if VisualSetters.AutoRight then VisualSetters.AutoRight(true) end
            elseif s=="right" then
                aplOn=true startAutoPlayLeft()
                if VisualSetters.AutoLeft then VisualSetters.AutoLeft(true) end
            end
        end
        if ct>COUNTDOWN_TARGET+2 then G_countdownActive=false if conn then conn:Disconnect() conn=nil end end
    end)
end

workspace.ChildAdded:Connect(function(child)
    if child.Name=="Countdown" and child:IsA("Sound") then monitorCountdown(child) end
end)
do local ex=workspace:FindFirstChild("Countdown") if ex and ex:IsA("Sound") then monitorCountdown(ex) end end

-- ── Main speed loop ───────────────────────────────────────────
local _mainSpeedTick=0
RunService.Heartbeat:Connect(function()
    if not gChar or not gHum or not gHrp then return end
    if desyncActive then return end
    _mainSpeedTick=_mainSpeedTick+1 if _mainSpeedTick%2~=0 then return end
    if not autoBatToggled and not aplOn and not aprOn then
        local md=gHum.MoveDirection
        if md.Magnitude>0.1 then
            local spd=slowDownEnabled and SLOW_SPEED or NORMAL_SPEED
            gHrp.AssemblyLinearVelocity=Vector3.new(md.X*spd,gHrp.AssemblyLinearVelocity.Y,md.Z*spd)
        end
    end
end)

-- ── Desync velocity loop (WORKS with BodyVelocity desync) ─────
local _dsTick=0
RunService.Heartbeat:Connect(function()
    if not desyncActive then return end
    if not gChar or not gHum or not gHrp then return end
    if autoBatToggled or aplOn or aprOn then return end
    _dsTick=_dsTick+1 if _dsTick%2~=0 then return end
    local targetSpd = slowDownEnabled and DS_CARRY_SPEED or DS_NORMAL_SPEED
    if gHum.WalkSpeed ~= targetSpd then gHum.WalkSpeed = targetSpd end
end)

local function hookCarryOnTools(c)
    if _carryToolConn then _carryToolConn:Disconnect() _carryToolConn=nil end
    _carryToolConn = c.ChildAdded:Connect(function(obj)
        if obj:IsA("Tool") then
            task.wait(0.05)
            local h = c:FindFirstChildOfClass("Humanoid")
            if not h then return end
            if desyncActive then
                local spd = DS_CARRY_SPEED
                h.WalkSpeed = spd
            elseif slowDownEnabled then
                h.WalkSpeed = SLOW_SPEED
            end
        end
    end)
end

local _carryToolConn = nil

local function setupChar(c)
    gChar=c gHum=c:WaitForChild("Humanoid",5) gHrp=c:WaitForChild("HumanoidRootPart",5)
    if not gHum or not gHrp then return end
    stopAntiRagdoll() startAntiRagdoll()
    antiRagdollEnabled=true
    if not autoStealConn then startAutoSteal() end
    autoStealEnabled=true
    if autoBatToggled then stopBatAimbot() startBatAimbot() end
    if lagModeEnabled then startLagMode() end
    G_lastKnownHealth=gHum.Health
    G_tpReturnCooldown=false
    task.wait(0.1)
    if desyncActive then
        gHum.WalkSpeed = slowDownEnabled and DS_CARRY_SPEED or DS_NORMAL_SPEED
    elseif slowDownEnabled then
        gHum.WalkSpeed = SLOW_SPEED
    end
    hookCarryOnTools(c)
end
if player.Character then setupChar(player.Character) end
player.CharacterAdded:Connect(function(c) task.wait(0.5) setupChar(c) end)

local _carryHbTick=0
RunService.Heartbeat:Connect(function()
    if not slowDownEnabled or not gHum then return end
    if desyncActive then return end
    _carryHbTick=_carryHbTick+1 if _carryHbTick%5~=0 then return end
    if gHum.WalkSpeed ~= SLOW_SPEED and gHum.WalkSpeed > 0 then
        gHum.WalkSpeed = SLOW_SPEED
    end
end)

-- ── SOLO DUELS USER BILLBOARD ─────────────────────────────────
local J5_TAG = "SoloHubUser"

local function makeJ5Billboard(character)
    local head = character:FindFirstChild("Head")
    if not head then return end
    if head:FindFirstChild("J5UserBillboard") then return end
    local bb = Instance.new("BillboardGui")
    bb.Name = "J5UserBillboard"
    bb.Adornee = head
    bb.Size = UDim2.new(0, 180, 0, 28)
    bb.StudsOffset = Vector3.new(0, 4.2, 0)
    bb.AlwaysOnTop = true
    bb.ResetOnSpawn = false
    bb.Parent = head
    local lbl = Instance.new("TextLabel", bb)
    lbl.Size = UDim2.new(1, 0, 1, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = "using solo"
    lbl.TextColor3 = Color3.fromRGB(80, 255, 80)
    lbl.TextStrokeColor3 = Color3.fromRGB(0, 80, 0)
    lbl.TextStrokeTransparency = 0.3
    lbl.Font = Enum.Font.GothamBold
    lbl.TextSize = 13
end

local function removeJ5Billboard(character)
    local head = character:FindFirstChild("Head")
    if not head then return end
    local bb = head:FindFirstChild("J5UserBillboard")
    if bb then bb:Destroy() end
end

local function tagMyCharacter(c)
    pcall(function()
        local tag = Instance.new("StringValue")
        tag.Name = J5_TAG
        tag.Value = player.Name
        tag.Parent = c
    end)
end
if player.Character then
    tagMyCharacter(player.Character)
    makeJ5Billboard(player.Character)
end
player.CharacterAdded:Connect(function(c)
    task.wait(0.5)
    tagMyCharacter(c)
    makeJ5Billboard(c)
end)

local function scanPlayer(p)
    if p == player then return end
    local c = p.Character
    if c and c:FindFirstChild(J5_TAG) then
        makeJ5Billboard(c)
    end
    p.CharacterAdded:Connect(function(newC)
        task.wait(0.6)
        if newC:FindFirstChild(J5_TAG) then
            makeJ5Billboard(newC)
        end
        newC.ChildAdded:Connect(function(child)
            if child.Name == J5_TAG then
                makeJ5Billboard(newC)
            end
        end)
    end)
    if c then
        c.ChildAdded:Connect(function(child)
            if child.Name == J5_TAG then
                makeJ5Billboard(c)
            end
        end)
        c.ChildRemoved:Connect(function(child)
            if child.Name == J5_TAG then
                removeJ5Billboard(c)
            end
        end)
    end
end

for _, p in ipairs(Players:GetPlayers()) do
    task.spawn(scanPlayer, p)
end
Players.PlayerAdded:Connect(function(p)
    task.spawn(scanPlayer, p)
end)

-- ═══════════════════════════════════════════════════════════════
--  GUI (Solo Duels Hub)
-- ═══════════════════════════════════════════════════════════════
local gui=Instance.new("ScreenGui")
gui.Name="SoloDuelsGUI" gui.ResetOnSpawn=false gui.DisplayOrder=10 gui.IgnoreGuiInset=true
gui.Parent=player:WaitForChild("PlayerGui")

local VisualSetters={}

-- ── FLOATING AUTO PLAY BUTTON + ⚙️ side-picker emoji ──────────
do
    local apLado      = nil
    local apAtivo     = false
    local apWasActive = false

    local fAuto = Instance.new("TextButton", gui)
    fAuto.Name = "SoloAutoPlayFloat"
    fAuto.BorderSizePixel = 0
    fAuto.BackgroundColor3 = MB_BG
    fAuto.Size = UDim2.new(0, 110, 0, 44)
    fAuto.Position = UDim2.new(0, 129, 0, 136)
    fAuto.Active = true
    fAuto.ZIndex = 20
    fAuto.Text = "⚡ AUTO\nPLAY"
    fAuto.Font = Enum.Font.GothamBlack
    fAuto.TextSize = 11
    fAuto.TextColor3 = MB_DIM
    fAuto.AutoButtonColor = false
    Instance.new("UICorner", fAuto).CornerRadius = UDim.new(0, 16)
    local fStroke = Instance.new("UIStroke", fAuto)
    fStroke.Thickness = 2
    fStroke.Color = MB_WHITE
    fStroke.Transparency = 0.55

    local gearBtn = Instance.new("TextButton", gui)
    gearBtn.Name = "SoloAPGear"
    gearBtn.BorderSizePixel = 0
    gearBtn.BackgroundColor3 = MB_BG
    gearBtn.Size = UDim2.new(0, 30, 0, 30)
    gearBtn.Position = UDim2.new(0, 129 + 110 + 6, 0, 136 + 7)
    gearBtn.ZIndex = 20
    gearBtn.Text = "⚙️"
    gearBtn.Font = Enum.Font.GothamBold
    gearBtn.TextSize = 16
    gearBtn.AutoButtonColor = false
    Instance.new("UICorner", gearBtn).CornerRadius = UDim.new(0, 10)
    local gearStroke = Instance.new("UIStroke", gearBtn)
    gearStroke.Thickness = 1.5
    gearStroke.Color = MB_WHITE
    gearStroke.Transparency = 0.6

    local picker = Instance.new("Frame", gui)
    picker.Name = "SoloAPPicker"
    picker.BorderSizePixel = 0
    picker.BackgroundColor3 = Color3.fromRGB(5, 20, 5)
    picker.Size = UDim2.new(0, 130, 0, 80)
    picker.Position = UDim2.new(0, 129 + 110 + 6 - 50, 0, 136 + 44)
    picker.ZIndex = 200
    picker.Visible = false
    Instance.new("UICorner", picker).CornerRadius = UDim.new(0, 14)
    local pickStroke = Instance.new("UIStroke", picker)
    pickStroke.Thickness = 1.8; pickStroke.Color = MB_WHITE; pickStroke.Transparency = 0.4

    local pickTitle = Instance.new("TextLabel", picker)
    pickTitle.Size = UDim2.new(1,0,0,22)
    pickTitle.Position = UDim2.new(0,0,0,4)
    pickTitle.BackgroundTransparency = 1
    pickTitle.Text = "Choose Side"
    pickTitle.Font = Enum.Font.GothamBlack
    pickTitle.TextSize = 11
    pickTitle.TextColor3 = Color3.fromRGB(220,220,220)
    pickTitle.ZIndex = 201

    local function makeSideBtn(labelTxt, xPos)
        local b = Instance.new("TextButton", picker)
        b.Size = UDim2.new(0, 54, 0, 30)
        b.Position = UDim2.new(0, xPos, 0, 30)
        b.BackgroundColor3 = MB_BG
        b.BorderSizePixel = 0
        b.Text = labelTxt
        b.Font = Enum.Font.GothamBlack
        b.TextSize = 11
        b.TextColor3 = MB_DIM
        b.AutoButtonColor = false
        b.ZIndex = 202
        Instance.new("UICorner", b).CornerRadius = UDim.new(0, 10)
        local s = Instance.new("UIStroke", b)
        s.Thickness = 1.5; s.Color = MB_WHITE; s.Transparency = 0.55
        return b, s
    end
    local btnSL, sSL = makeSideBtn("LEFT",  8)
    local btnSR, sSR = makeSideBtn("RIGHT", 68)

    local function apUpdateVisual()
        if apAtivo then
            fStroke.Transparency = 0.15
            TweenService:Create(fAuto,TweenInfo.new(0.15),{BackgroundColor3=MB_ACTIVE}):Play()
        else
            fStroke.Transparency = 0.55
            TweenService:Create(fAuto,TweenInfo.new(0.15),{BackgroundColor3=MB_BG}):Play()
        end
        sSL.Transparency = (apLado=="LEFT")  and 0.0 or 0.55
        sSR.Transparency = (apLado=="RIGHT") and 0.0 or 0.55
        btnSL.TextColor3 = (apLado=="LEFT")  and Color3.fromRGB(100,255,100) or MB_DIM
        btnSR.TextColor3 = (apLado=="RIGHT") and Color3.fromRGB(100,255,100) or MB_DIM
        if apLado == "LEFT" then
            gearBtn.Text = "◀"
        elseif apLado == "RIGHT" then
            gearBtn.Text = "▶"
        else
            gearBtn.Text = "⚙️"
        end
    end
    apUpdateVisual()

    local pickerOpen = false
    local function openPicker()
        pickerOpen = true
        picker.Visible = true
        picker.Size = UDim2.new(0,130,0,0)
        TweenService:Create(picker, TweenInfo.new(0.18, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
            {Size=UDim2.new(0,130,0,80)}):Play()
        TweenService:Create(gearStroke,TweenInfo.new(0.12),{Transparency=0.1}):Play()
    end
    local function closePicker()
        pickerOpen = false
        TweenService:Create(picker, TweenInfo.new(0.14, Enum.EasingStyle.Quad, Enum.EasingDirection.In),
            {Size=UDim2.new(0,130,0,0)}):Play()
        task.delay(0.15, function() picker.Visible=false end)
        TweenService:Create(gearStroke,TweenInfo.new(0.12),{Transparency=0.6}):Play()
    end

    local function apStop()
        apAtivo=false; apWasActive=false
        if aplOn then aplOn=false if aplConn then aplConn:Disconnect() aplConn=nil end aplPhase=1
            if VisualSetters.AutoLeft then VisualSetters.AutoLeft(false) end end
        if aprOn then aprOn=false if aprConn then aprConn:Disconnect() aprConn=nil end aprPhase=1
            if VisualSetters.AutoRight then VisualSetters.AutoRight(false) end end
        local ch=player.Character
        if ch then local hum=ch:FindFirstChildOfClass("Humanoid") if hum then hum:Move(Vector3.zero) end end
        apUpdateVisual()
    end
    local function apStart()
        if not apLado then return end
        apAtivo=true; apWasActive=true
        if apLado=="LEFT" then
            if aprOn then aprOn=false if aprConn then aprConn:Disconnect() aprConn=nil end aprPhase=1
                if VisualSetters.AutoRight then VisualSetters.AutoRight(false) end end
            aplOn=true; startAutoPlayLeft()
            if VisualSetters.AutoLeft then VisualSetters.AutoLeft(true) end
        else
            if aplOn then aplOn=false if aplConn then aplConn:Disconnect() aplConn=nil end aplPhase=1
                if VisualSetters.AutoLeft then VisualSetters.AutoLeft(false) end end
            aprOn=true; startAutoPlayRight()
            if VisualSetters.AutoRight then VisualSetters.AutoRight(true) end
        end
        apUpdateVisual()
    end

    gearBtn.MouseButton1Click:Connect(function()
        playTick()
        if pickerOpen then closePicker() else openPicker() end
    end)

    btnSL.MouseButton1Click:Connect(function()
        playTick()
        apLado="LEFT"; apUpdateVisual(); closePicker()
    end)
    btnSR.MouseButton1Click:Connect(function()
        playTick()
        apLado="RIGHT"; apUpdateVisual(); closePicker()
    end)

    local apDragging=false; local apDragStart; local apFrameStart; local apDragMoved=false
    fAuto.InputBegan:Connect(function(inp)
        if inp.UserInputType==Enum.UserInputType.MouseButton1 or inp.UserInputType==Enum.UserInputType.Touch then
            apDragging=true; apDragMoved=false
            apDragStart=inp.Position; apFrameStart=fAuto.Position
        end
    end)
    fAuto.InputEnded:Connect(function(inp)
        if inp.UserInputType==Enum.UserInputType.MouseButton1 or inp.UserInputType==Enum.UserInputType.Touch then
            apDragging=false
        end
    end)
    UserInputService.InputChanged:Connect(function(inp)
        if apDragging and not _sidePanelLocked and (inp.UserInputType==Enum.UserInputType.MouseMovement or inp.UserInputType==Enum.UserInputType.Touch) then
            local d=inp.Position-apDragStart
            if d.Magnitude>6 then
                apDragMoved=true
                local newPos=UDim2.new(apFrameStart.X.Scale,apFrameStart.X.Offset+d.X,apFrameStart.Y.Scale,apFrameStart.Y.Offset+d.Y)
                fAuto.Position=newPos
                gearBtn.Position=UDim2.new(newPos.X.Scale,newPos.X.Offset+110+6,newPos.Y.Scale,newPos.Y.Offset+7)
                picker.Position=UDim2.new(newPos.X.Scale,newPos.X.Offset+110+6-50,newPos.Y.Scale,newPos.Y.Offset+44)
            end
        end
    end)
    fAuto.MouseButton1Click:Connect(function()
        if apDragMoved then return end
        if not apLado then
            playTick(); openPicker(); return
        end
        playTick()
        if apAtivo then apStop() else apStart() end
    end)
    fAuto.MouseEnter:Connect(function()
        if not apAtivo then TweenService:Create(fAuto,TweenInfo.new(0.12),{BackgroundColor3=Color3.fromRGB(10,30,10)}):Play() end
    end)
    fAuto.MouseLeave:Connect(function()
        if not apAtivo then TweenService:Create(fAuto,TweenInfo.new(0.12),{BackgroundColor3=MB_BG}):Play() end
    end)

    UserInputService.InputBegan:Connect(function(inp,gpe)
        if gpe then return end
        local kb=Keybinds and Keybinds["AutoPlayTimer"]
        if kb and inp.KeyCode==kb and apLado then
            playTick()
            if apAtivo then apStop() else apStart() end
        end
    end)

    player.CharacterAdded:Connect(function()
        aplOn=false; aprOn=false; apAtivo=false; apWasActive=false; apUpdateVisual()
    end)
    player.CharacterRemoving:Connect(function()
        aplOn=false; aprOn=false; apAtivo=false; apUpdateVisual()
    end)
end

local function createToggle(parent,labelText,enabledKey,defaultOn,order,onToggle)
    local row=Instance.new("Frame",parent)
    row.Size=UDim2.new(0.96,0,0,32) row.BackgroundColor3=C_CARD
    row.BackgroundTransparency=0.3 row.BorderSizePixel=0
    row.LayoutOrder=order row.ZIndex=7
    Instance.new("UICorner",row).CornerRadius=UDim.new(0,8)
    Instance.new("UIStroke",row).Color=C_DIM
    local rowStroke=row:FindFirstChildOfClass("UIStroke")
    rowStroke.Thickness=1 rowStroke.Transparency=0.6

    local label=Instance.new("TextLabel",row)
    label.Size=UDim2.new(0.58,0,1,0) label.Position=UDim2.new(0.04,0,0,0)
    label.BackgroundTransparency=1 label.Text=labelText
    label.TextColor3=Color3.new(1,1,1) label.Font=Enum.Font.GothamBold label.TextSize=11
    label.TextXAlignment=Enum.TextXAlignment.Left label.ZIndex=8

    local kbBind=Keybinds[enabledKey]
    local kbName=kbBind and kbBind.Name or "–"
    local kbBtn=Instance.new("TextButton",row)
    kbBtn.Size=UDim2.new(0,28,0,14) kbBtn.Position=UDim2.new(1,-76,0.5,-7)
    kbBtn.BackgroundColor3=Color3.fromRGB(5,22,5) kbBtn.BorderSizePixel=0
    kbBtn.Text=kbName kbBtn.TextColor3=C_DIM kbBtn.Font=Enum.Font.GothamBold
    kbBtn.TextSize=7 kbBtn.ZIndex=10 kbBtn.AutoButtonColor=false
    Instance.new("UICorner",kbBtn).CornerRadius=UDim.new(0,4)
    local kbStroke=Instance.new("UIStroke",kbBtn) kbStroke.Color=C_GREEN kbStroke.Thickness=1 kbStroke.Transparency=0.5
    kbBtn.MouseButton1Click:Connect(function()
        if changingKeybind==enabledKey then
            changingKeybind=nil kbBtn.Text=Keybinds[enabledKey] and Keybinds[enabledKey].Name or "–"
            TweenService:Create(kbBtn,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(5,22,5),TextColor3=C_DIM}):Play()
            return
        end
        changingKeybind=enabledKey
        kbBtn.Text="..."
        TweenService:Create(kbBtn,TweenInfo.new(0.15),{BackgroundColor3=C_GREEN,TextColor3=C_WHITE}):Play()
    end)
    if enabledKey then
        UserInputService.InputBegan:Connect(function(inp,gpe)
            if changingKeybind~=enabledKey then return end
            if inp.UserInputType~=Enum.UserInputType.Keyboard then return end
            Keybinds[enabledKey]=inp.KeyCode
            kbBtn.Text=inp.KeyCode.Name
            TweenService:Create(kbBtn,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(5,22,5),TextColor3=C_DIM}):Play()
            changingKeybind=nil
        end)
    end

    local ctrlName=ControllerBinds and ControllerBinds[enabledKey] or nil
    if ctrlName then
        local cPill=Instance.new("TextButton",row)
        cPill.Size=UDim2.new(0,22,0,14) cPill.Position=UDim2.new(1,-104,0.5,-7)
        cPill.BackgroundColor3=Color3.fromRGB(5,22,5) cPill.BorderSizePixel=0
        cPill.Text=ctrlName cPill.TextColor3=Color3.fromRGB(30,200,30)
        cPill.Font=Enum.Font.GothamBold cPill.TextSize=7 cPill.ZIndex=10 cPill.AutoButtonColor=false
        Instance.new("UICorner",cPill).CornerRadius=UDim.new(0,4)
        local cpStroke=Instance.new("UIStroke",cPill) cpStroke.Color=Color3.fromRGB(30,180,30) cpStroke.Thickness=1 cpStroke.Transparency=0.5
        local _ctrlChanging=false
        cPill.MouseButton1Click:Connect(function()
            if _ctrlChanging then
                _ctrlChanging=false cPill.Text=ControllerBinds[enabledKey] or ctrlName
                TweenService:Create(cPill,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(5,22,5),TextColor3=Color3.fromRGB(30,200,30)}):Play()
                return
            end
            _ctrlChanging=true cPill.Text="..."
            TweenService:Create(cPill,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(10,35,10),TextColor3=Color3.fromRGB(200,200,200)}):Play()
        end)
        UserInputService.InputBegan:Connect(function(inp,gpe)
            if not _ctrlChanging then return end
            local ut=inp.UserInputType
            if ut~=Enum.UserInputType.Gamepad1 and ut~=Enum.UserInputType.Gamepad2 then return end
            local kc=inp.KeyCode
            local skip={Enum.KeyCode.Thumbstick1,Enum.KeyCode.Thumbstick2}
            for _,s in ipairs(skip) do if kc==s then return end end
            local nm={ButtonA="A",ButtonB="B",ButtonX="X",ButtonY="Y",ButtonCross="✕",ButtonCircle="○",ButtonSquare="□",ButtonTriangle="△",ButtonL1="L1",ButtonR1="R1",ButtonL2="L2",ButtonR2="R2",ButtonL3="L3",ButtonR3="R3",DPadUp="D↑",DPadDown="D↓",DPadLeft="D←",DPadRight="D→",ButtonStart="Start",ButtonSelect="Sel",ButtonBack="Back"}
            local dn=nm[kc.Name] or kc.Name
            ControllerBinds[enabledKey]=dn cPill.Text=dn _ctrlChanging=false
            TweenService:Create(cPill,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(5,22,5),TextColor3=Color3.fromRGB(30,200,30)}):Play()
        end)
    end

    local swBg=Instance.new("Frame",row)
    swBg.Size=UDim2.new(0,38,0,18) swBg.Position=UDim2.new(1,-44,0.5,-9)
    swBg.BackgroundColor3=defaultOn and Color3.fromRGB(30,200,30) or C_SW_OFF
    swBg.BorderSizePixel=0 swBg.ZIndex=8
    Instance.new("UICorner",swBg).CornerRadius=UDim.new(1,0)
    local swDot=Instance.new("Frame",swBg)
    swDot.Size=UDim2.new(0,14,0,14)
    swDot.Position=defaultOn and UDim2.new(0.55,0,0.1,0) or UDim2.new(0.08,0,0.1,0)
    swDot.BackgroundColor3=Color3.new(1,1,1) swDot.BorderSizePixel=0 swDot.ZIndex=9
    Instance.new("UICorner",swDot).CornerRadius=UDim.new(1,0)

    local isOn=defaultOn
    local function setVisual(state)
        isOn=state
        TweenService:Create(swBg,TweenInfo.new(0.18),{BackgroundColor3=isOn and Color3.fromRGB(30,200,30) or C_SW_OFF}):Play()
        TweenService:Create(swDot,TweenInfo.new(0.18),{Position=isOn and UDim2.new(0.55,0,0.1,0) or UDim2.new(0.08,0,0.1,0)}):Play()
        if mobBtnRefs[enabledKey] then
            local mb=mobBtnRefs[enabledKey]
            mb.Visible=true
            TweenService:Create(mb,TweenInfo.new(0.15),{BackgroundColor3=state and MB_ACTIVE or MB_BG}):Play()
        end
    end
    if enabledKey then VisualSetters[enabledKey]=setVisual end
    task.defer(function()
        if mobBtnRefs[enabledKey] then
            local mb=mobBtnRefs[enabledKey]
            mb.Visible=true
            mb.BackgroundColor3=defaultOn and MB_ACTIVE or MB_BG
        end
    end)

    local btn=Instance.new("TextButton",row)
    btn.Size=UDim2.new(1,0,1,0) btn.BackgroundTransparency=1 btn.Text="" btn.ZIndex=9
    btn.MouseButton1Click:Connect(function()
        if changingKeybind then return end
        playTick() isOn=not isOn setVisual(isOn)
        if onToggle then onToggle(isOn) end
    end)

    return setVisual,swBg,swDot
end

local function createSection(parent,text,order)
    local s=Instance.new("Frame",parent)
    s.Size=UDim2.new(1,0,0,18) s.BackgroundTransparency=1 s.LayoutOrder=order s.ZIndex=3
    local lbl=Instance.new("TextLabel",s)
    lbl.Size=UDim2.new(1,-8,1,0) lbl.Position=UDim2.new(0,4,0,0)
    lbl.BackgroundTransparency=1 lbl.Text=text:upper()
    lbl.TextColor3=Color3.fromRGB(130,130,130) lbl.Font=Enum.Font.GothamBold lbl.TextSize=9
    lbl.TextXAlignment=Enum.TextXAlignment.Left lbl.ZIndex=4
    local line=Instance.new("Frame",s)
    line.Size=UDim2.new(1,0,0,1) line.Position=UDim2.new(0,0,1,0)
    line.BackgroundColor3=Color3.fromRGB(20,80,20) line.BackgroundTransparency=0.4 line.BorderSizePixel=0 line.ZIndex=3
end

local BASE_W,BASE_H=270,500
local main=Instance.new("CanvasGroup")
main.Name="Main" main.Size=UDim2.new(0,BASE_W,0,BASE_H)
main.Position=UDim2.new(0,10,0,70)
_registerBtn("mainPanel", main) main.BackgroundColor3=C_BG
main.BorderSizePixel=0 main.Active=true main.ClipsDescendants=false
main.GroupTransparency=0
main.Visible=false main.Parent=gui
Instance.new("UICorner",main).CornerRadius=UDim.new(0,28)
local mainStroke=Instance.new("UIStroke",main)
mainStroke.Thickness=2 mainStroke.Color=Color3.fromRGB(15,15,15)
task.spawn(function()
    while mainStroke and mainStroke.Parent do
        TweenService:Create(mainStroke, TweenInfo.new(1.4, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut),
            {Color=Color3.fromRGB(30,200,30), Thickness=2.5}):Play()
        task.wait(1.4)
        TweenService:Create(mainStroke, TweenInfo.new(1.4, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut),
            {Color=Color3.fromRGB(15,15,15), Thickness=1.8}):Play()
        task.wait(1.4)
    end
end)
do local sc=Instance.new("UIScale",main) sc.Scale=UI_SCALE end
gui.AutoLocalize=false

do
    local drag,ds,sp=false
    main.InputBegan:Connect(function(inp)
        if inp.UserInputType==Enum.UserInputType.MouseButton1 or inp.UserInputType==Enum.UserInputType.Touch then
            drag=true ds=inp.Position sp=main.Position
            inp.Changed:Connect(function() if inp.UserInputState==Enum.UserInputState.End then drag=false end end)
        end
    end)
    UserInputService.InputChanged:Connect(function(inp)
        if drag and (inp.UserInputType==Enum.UserInputType.MouseMovement or inp.UserInputType==Enum.UserInputType.Touch) then
            local d=inp.Position-ds main.Position=UDim2.new(sp.X.Scale,sp.X.Offset+d.X,sp.Y.Scale,sp.Y.Offset+d.Y)
        end
    end)
end

local header=Instance.new("Frame",main)
header.Size=UDim2.new(1,0,0,64) header.BackgroundColor3=C_BG header.BorderSizePixel=0 header.ZIndex=3
Instance.new("UICorner",header).CornerRadius=UDim.new(0,28)
local hFix=Instance.new("Frame",header) hFix.Size=UDim2.new(1,0,0,20) hFix.Position=UDim2.new(0,0,1,-20)
hFix.BackgroundColor3=C_BG hFix.BorderSizePixel=0 hFix.ZIndex=2
local headerTitle=Instance.new("TextLabel",header)
headerTitle.Size=UDim2.new(1,0,0,28) headerTitle.Position=UDim2.new(0,0,0,4)
headerTitle.BackgroundTransparency=1 headerTitle.Text="SOLO DUELS"
headerTitle.TextColor3=Color3.fromRGB(50,230,50)
headerTitle.Font=Enum.Font.GothamBlack headerTitle.TextSize=20
headerTitle.TextXAlignment=Enum.TextXAlignment.Center headerTitle.ZIndex=4
local headerDiscord=Instance.new("TextLabel",header)
headerDiscord.Size=UDim2.new(1,0,0,14) headerDiscord.Position=UDim2.new(0,0,0,32)
headerDiscord.BackgroundTransparency=1 headerDiscord.Text="https://discord.gg/vhMn5vZZkc"
headerDiscord.TextColor3=Color3.fromRGB(40,160,40)
headerDiscord.Font=Enum.Font.GothamBold headerDiscord.TextSize=9
headerDiscord.TextXAlignment=Enum.TextXAlignment.Center headerDiscord.ZIndex=4
do
    local t=0 local dir=1
    RunService.Heartbeat:Connect(function(dt)
        if not headerTitle or not headerTitle.Parent then return end
        t=t+dt*0.7*dir
        if t>=1 then t=1 dir=-1 elseif t<=0 then t=0 dir=1 end
        local v=math.floor(t*255)
        headerTitle.TextColor3=Color3.fromRGB(v,v,v)
    end)
end

local tabContainer=Instance.new("Frame",header)
tabContainer.Size=UDim2.new(1,-12,0,24) tabContainer.Position=UDim2.new(0,6,0,38)
tabContainer.BackgroundColor3=C_TABBG tabContainer.BorderSizePixel=0 tabContainer.ZIndex=4
Instance.new("UICorner",tabContainer).CornerRadius=UDim.new(0,6)
local tabLayout=Instance.new("UIListLayout",tabContainer)
tabLayout.FillDirection=Enum.FillDirection.Horizontal
tabLayout.HorizontalAlignment=Enum.HorizontalAlignment.Center
tabLayout.SortOrder=Enum.SortOrder.LayoutOrder tabLayout.Padding=UDim.new(0,2)
local currentTab="Combat" local tabPages={}

local function createTab(tabName,order)
    local tabBtn=Instance.new("TextButton",tabContainer)
    tabBtn.Size=UDim2.new(0.25,0,1,0)
    tabBtn.BackgroundTransparency=1
    tabBtn.BorderSizePixel=0 tabBtn.Text=tabName
    tabBtn.TextColor3=currentTab==tabName and Color3.fromRGB(30,200,30) or Color3.fromRGB(90,90,90)
    tabBtn.Font=Enum.Font.GothamBold tabBtn.TextSize=9 tabBtn.LayoutOrder=order tabBtn.ZIndex=5
    tabBtn.MouseButton1Click:Connect(function()
        playTick() currentTab=tabName
        for _,btn in ipairs(tabContainer:GetChildren()) do
            if btn:IsA("TextButton") then
                TweenService:Create(btn,TweenInfo.new(0.15),{
                    TextColor3=btn.Text==tabName and Color3.fromRGB(30,200,30) or Color3.fromRGB(90,90,90)
                }):Play()
            end
        end
        for pn,page in pairs(tabPages) do page.Visible=pn==tabName end
    end)
    return tabBtn
end
createTab("Combat",1) createTab("Movement",2) createTab("Auto",3) createTab("Visual",4)

local scroll=Instance.new("ScrollingFrame",main)
scroll.Size=UDim2.new(1,-8,1,-72) scroll.Position=UDim2.new(0,4,0,66)
scroll.BackgroundTransparency=1 scroll.BorderSizePixel=0 scroll.ScrollBarThickness=3
scroll.ScrollBarImageColor3=C_GREEN scroll.AutomaticCanvasSize=Enum.AutomaticSize.Y
scroll.CanvasSize=UDim2.new(0,0,0,0) scroll.ZIndex=2

local function createTabPage(tabName)
    local page=Instance.new("Frame",scroll)
    page.Size=UDim2.new(1,0,1,0) page.BackgroundTransparency=1
    page.Visible=tabName=="Combat" page.ZIndex=2
    local pl=Instance.new("UIListLayout",page) pl.Padding=UDim.new(0,3) pl.SortOrder=Enum.SortOrder.LayoutOrder
    tabPages[tabName]=page return page
end
local combatPage=createTabPage("Combat")
local movementPage=createTabPage("Movement")
local autoPage=createTabPage("Auto")
local visualPage=createTabPage("Visual")

local userInfoBar=Instance.new("Frame",main)
userInfoBar.Size=UDim2.new(1,-20,0,36) userInfoBar.Position=UDim2.new(0,10,1,-46)
userInfoBar.BackgroundColor3=C_CARD userInfoBar.BorderSizePixel=0 userInfoBar.ZIndex=3
Instance.new("UICorner",userInfoBar).CornerRadius=UDim.new(0,10)
local userStroke=Instance.new("UIStroke",userInfoBar) userStroke.Color=C_GREEN userStroke.Thickness=1 userStroke.Transparency=0.7
local dotDot=Instance.new("Frame",userInfoBar)
dotDot.Size=UDim2.new(0,8,0,8) dotDot.Position=UDim2.new(0,10,0.5,-4)
dotDot.BackgroundColor3=Color3.fromRGB(30, 220, 30) dotDot.BorderSizePixel=0 dotDot.ZIndex=5
Instance.new("UICorner",dotDot).CornerRadius=UDim.new(1,0)
local usernameLabel=Instance.new("TextLabel",userInfoBar)
usernameLabel.Size=UDim2.new(1,-30,1,0) usernameLabel.Position=UDim2.new(0,24,0,0)
usernameLabel.BackgroundTransparency=1 usernameLabel.Text=player.Name
usernameLabel.TextColor3=C_WHITE usernameLabel.Font=Enum.Font.GothamBlack
usernameLabel.TextSize=12 usernameLabel.TextXAlignment=Enum.TextXAlignment.Left
usernameLabel.TextTruncate=Enum.TextTruncate.AtEnd usernameLabel.ZIndex=4

local _speedBillboard = nil
local function setupSpeedDisplay(c)
    if _speedBillboard then _speedBillboard:Destroy() _speedBillboard=nil end
    local head = c:WaitForChild("Head",5) if not head then return end
    local bb = Instance.new("BillboardGui")
    bb.Name = "J5SpeedDisplay"
    bb.Adornee = head
    bb.Size = UDim2.new(0,100,0,22)
    bb.StudsOffset = Vector3.new(0,5.8,0)
    bb.AlwaysOnTop = true
    bb.ResetOnSpawn = false
    bb.Parent = head
    _speedBillboard = bb
    local lbl = Instance.new("TextLabel",bb)
    lbl.Size = UDim2.new(1,0,1,0) lbl.BackgroundTransparency=1
    lbl.Text = "0 stud/s" lbl.TextColor3 = Color3.fromRGB(30, 220, 30)
    lbl.Font = Enum.Font.GothamBlack lbl.TextSize = 11
    local txtStroke = Instance.new("UIStroke",lbl)
    txtStroke.Color = Color3.fromRGB(0,0,0) txtStroke.Thickness = 1.5 txtStroke.Transparency = 0.4
    local _hbCount=0
    RunService.Heartbeat:Connect(function()
        if not _speedBillboard or not _speedBillboard.Parent then return end
        _hbCount=_hbCount+1 if _hbCount%3~=0 then return end
        local hrp = c:FindFirstChild("HumanoidRootPart")
        if not hrp then lbl.Text="0 stud/s" return end
        local vel = hrp.AssemblyLinearVelocity
        local hspd = math.floor(Vector3.new(vel.X,0,vel.Z).Magnitude + 0.5)
        local spd = slowDownEnabled and SLOW_SPEED or NORMAL_SPEED
        local col = hspd >= spd*0.9 and Color3.fromRGB(80, 255, 80) or Color3.fromRGB(30, 200, 30)
        lbl.Text = hspd.." stud/s"
        lbl.TextColor3 = col
    end)
end
if player.Character then task.spawn(function() setupSpeedDisplay(player.Character) end) end
player.CharacterAdded:Connect(function(c) task.wait(0.6) setupSpeedDisplay(c) end)

local order=0
local function O() order=order+1 return order end
createSection(combatPage,"Combat",O())
createToggle(combatPage,"Auto Bat","BatAimbot",false,O(),function(on)
    if on then
        if aplOn then stopAutoPlayLeft() if VisualSetters.AutoLeft then VisualSetters.AutoLeft(false) end end
        if aprOn then stopAutoPlayRight() if VisualSetters.AutoRight then VisualSetters.AutoRight(false) end end
        startBatAimbot()
    else stopBatAimbot() end
end)
createToggle(combatPage,"Anti Ragdoll","AntiRagdoll",true,O(),function(on)
    antiRagdollEnabled=on if on then startAntiRagdoll() else stopAntiRagdoll() end
end)
createToggle(combatPage,"Lag Mode","LagMode",false,O(),function(on)
    lagModeEnabled=on if on then startLagMode() else stopLagMode() end
end)
createToggle(combatPage,"Carry Mode","SlowDown",true,O(),function(on)
    slowDownEnabled=on
    local hum=getHum()
    if hum then
        if desyncActive then
            hum.WalkSpeed=on and DS_CARRY_SPEED or DS_NORMAL_SPEED
        else
            hum.WalkSpeed=on and SLOW_SPEED or NORMAL_SPEED
        end
    end
end)
createSection(combatPage,"Actions",O())

-- Drop button
do
    local container=Instance.new("Frame",combatPage)
    container.Size=UDim2.new(1,0,0,42) container.BackgroundColor3=C_CARD
    container.BorderSizePixel=0 container.LayoutOrder=O() container.ZIndex=3
    Instance.new("UICorner",container).CornerRadius=UDim.new(0,14)
    local cStroke=Instance.new("UIStroke",container) cStroke.Color=C_GREEN cStroke.Thickness=1 cStroke.Transparency=0.8
    local lbl=Instance.new("TextLabel",container)
    lbl.Size=UDim2.new(1,-120,0.5,0) lbl.Position=UDim2.new(0,10,0,0)
    lbl.BackgroundTransparency=1 lbl.Text="Drop" lbl.TextColor3=C_WHITE
    lbl.Font=Enum.Font.GothamBold lbl.TextSize=11 lbl.TextXAlignment=Enum.TextXAlignment.Left lbl.ZIndex=5
    local kbPill=Instance.new("TextLabel",container)
    kbPill.Size=UDim2.new(0,24,0,14) kbPill.Position=UDim2.new(0,10,0.5,1)
    kbPill.BackgroundColor3=Color3.fromRGB(30, 30, 30) kbPill.BorderSizePixel=0
    kbPill.Text="F3" kbPill.TextColor3=C_DIM kbPill.Font=Enum.Font.GothamBold kbPill.TextSize=8 kbPill.ZIndex=6
    Instance.new("UICorner",kbPill).CornerRadius=UDim.new(0,4)
    local _dropCtrlBind = "R2"
    local _dropCtrlChanging = false
    local cPill=Instance.new("TextButton",container)
    cPill.Size=UDim2.new(0,36,0,18) cPill.Position=UDim2.new(0,36,0.5,-1)
    cPill.BackgroundColor3=Color3.fromRGB(20, 20, 20) cPill.BorderSizePixel=0
    cPill.Text=_dropCtrlBind cPill.TextColor3=Color3.fromRGB(30, 220, 30)
    cPill.Font=Enum.Font.GothamBold cPill.TextSize=9 cPill.ZIndex=6 cPill.AutoButtonColor=false
    Instance.new("UICorner",cPill).CornerRadius=UDim.new(0,5)
    local cpStroke=Instance.new("UIStroke",cPill) cpStroke.Color=Color3.fromRGB(30, 180, 30) cpStroke.Thickness=1.5 cpStroke.Transparency=0.4
    cPill.MouseButton1Click:Connect(function()
        if _dropCtrlChanging then
            _dropCtrlChanging=false cPill.Text=_dropCtrlBind
            TweenService:Create(cPill,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(20, 20, 20),TextColor3=Color3.fromRGB(30, 220, 30)}):Play()
            return
        end
        _dropCtrlChanging=true cPill.Text="..."
        TweenService:Create(cPill,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(35, 35, 35),TextColor3=Color3.fromRGB(200, 200, 200)}):Play()
    end)
    UserInputService.InputBegan:Connect(function(inp,gpe)
        if not _dropCtrlChanging then return end
        local ut=inp.UserInputType
        if ut~=Enum.UserInputType.Gamepad1 and ut~=Enum.UserInputType.Gamepad2 then return end
        local skip={Enum.KeyCode.Thumbstick1,Enum.KeyCode.Thumbstick2}
        for _,s in ipairs(skip) do if inp.KeyCode==s then return end end
        local nm={ButtonA="A",ButtonB="B",ButtonX="X",ButtonY="Y",ButtonCross="✕",ButtonCircle="○",ButtonSquare="□",ButtonTriangle="△",ButtonL1="L1",ButtonR1="R1",ButtonL2="L2",ButtonR2="R2",ButtonL3="L3",ButtonR3="R3",DPadUp="D↑",DPadDown="D↓",DPadLeft="D←",DPadRight="D→",ButtonStart="Start",ButtonSelect="Sel"}
        _dropCtrlBind=nm[inp.KeyCode.Name] or inp.KeyCode.Name
        cPill.Text=_dropCtrlBind _dropCtrlChanging=false
        TweenService:Create(cPill,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(20, 20, 20),TextColor3=Color3.fromRGB(30, 220, 30)}):Play()
    end)
    local actBtn=Instance.new("TextButton",container)
    actBtn.Size=UDim2.new(0,48,0,22) actBtn.Position=UDim2.new(1,-54,0.5,-11)
    actBtn.BackgroundColor3=C_GREEN actBtn.BorderSizePixel=0
    actBtn.Text="USE" actBtn.TextColor3=C_BG actBtn.Font=Enum.Font.GothamBold actBtn.TextSize=9 actBtn.ZIndex=6
    Instance.new("UICorner",actBtn).CornerRadius=UDim.new(0,7)
    actBtn.MouseButton1Click:Connect(function()
        playTick()
        if setDropBrainrotVisual then setDropBrainrotVisual(true) end
        if dropMobileSetter then dropMobileSetter(true) end
        task.spawn(runDropBrainrot)
    end)
    actBtn.MouseEnter:Connect(function() TweenService:Create(cStroke,TweenInfo.new(0.15),{Transparency=0.3}):Play() end)
    actBtn.MouseLeave:Connect(function() TweenService:Create(cStroke,TweenInfo.new(0.15),{Transparency=0.8}):Play() end)
    setDropBrainrotVisual=function(state)
        TweenService:Create(actBtn,TweenInfo.new(0.15),{BackgroundColor3=state and C_WHITE or C_GREEN}):Play()
    end
end

-- TP Down button
do
    local container=Instance.new("Frame",combatPage)
    container.Size=UDim2.new(1,0,0,42) container.BackgroundColor3=C_CARD
    container.BorderSizePixel=0 container.LayoutOrder=O() container.ZIndex=3
    Instance.new("UICorner",container).CornerRadius=UDim.new(0,14)
    local cStroke=Instance.new("UIStroke",container) cStroke.Color=C_GREEN cStroke.Thickness=1 cStroke.Transparency=0.8
    local lbl=Instance.new("TextLabel",container)
    lbl.Size=UDim2.new(1,-120,0.5,0) lbl.Position=UDim2.new(0,10,0,0)
    lbl.BackgroundTransparency=1 lbl.Text="TP Down" lbl.TextColor3=C_WHITE
    lbl.Font=Enum.Font.GothamBold lbl.TextSize=11 lbl.TextXAlignment=Enum.TextXAlignment.Left lbl.ZIndex=5
    local kbPill=Instance.new("TextLabel",container)
    kbPill.Size=UDim2.new(0,20,0,14) kbPill.Position=UDim2.new(0,10,0.5,1)
    kbPill.BackgroundColor3=Color3.fromRGB(30, 30, 30) kbPill.BorderSizePixel=0
    kbPill.Text="G" kbPill.TextColor3=C_DIM kbPill.Font=Enum.Font.GothamBold kbPill.TextSize=8 kbPill.ZIndex=6
    Instance.new("UICorner",kbPill).CornerRadius=UDim.new(0,4)
    local _tpCtrlBind = "L2"
    local _tpCtrlChanging = false
    local cPill=Instance.new("TextButton",container)
    cPill.Size=UDim2.new(0,36,0,18) cPill.Position=UDim2.new(0,32,0.5,-1)
    cPill.BackgroundColor3=Color3.fromRGB(20, 20, 20) cPill.BorderSizePixel=0
    cPill.Text=_tpCtrlBind cPill.TextColor3=Color3.fromRGB(30, 220, 30)
    cPill.Font=Enum.Font.GothamBold cPill.TextSize=9 cPill.ZIndex=6 cPill.AutoButtonColor=false
    Instance.new("UICorner",cPill).CornerRadius=UDim.new(0,5)
    local cpStroke=Instance.new("UIStroke",cPill) cpStroke.Color=Color3.fromRGB(30, 180, 30) cpStroke.Thickness=1.5 cpStroke.Transparency=0.4
    cPill.MouseButton1Click:Connect(function()
        if _tpCtrlChanging then
            _tpCtrlChanging=false cPill.Text=_tpCtrlBind
            TweenService:Create(cPill,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(20, 20, 20),TextColor3=Color3.fromRGB(30, 220, 30)}):Play()
            return
        end
        _tpCtrlChanging=true cPill.Text="..."
        TweenService:Create(cPill,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(35, 35, 35),TextColor3=Color3.fromRGB(200, 200, 200)}):Play()
    end)
    UserInputService.InputBegan:Connect(function(inp,gpe)
        if not _tpCtrlChanging then return end
        local ut=inp.UserInputType
        if ut~=Enum.UserInputType.Gamepad1 and ut~=Enum.UserInputType.Gamepad2 then return end
        local skip={Enum.KeyCode.Thumbstick1,Enum.KeyCode.Thumbstick2}
        for _,s in ipairs(skip) do if inp.KeyCode==s then return end end
        local nm={ButtonA="A",ButtonB="B",ButtonX="X",ButtonY="Y",ButtonCross="✕",ButtonCircle="○",ButtonSquare="□",ButtonTriangle="△",ButtonL1="L1",ButtonR1="R1",ButtonL2="L2",ButtonR2="R2",ButtonL3="L3",ButtonR3="R3",DPadUp="D↑",DPadDown="D↓",DPadLeft="D←",DPadRight="D→",ButtonStart="Start",ButtonSelect="Sel"}
        _tpCtrlBind=nm[inp.KeyCode.Name] or inp.KeyCode.Name
        cPill.Text=_tpCtrlBind _tpCtrlChanging=false
        TweenService:Create(cPill,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(20, 20, 20),TextColor3=Color3.fromRGB(30, 220, 30)}):Play()
    end)
    local actBtn=Instance.new("TextButton",container)
    actBtn.Size=UDim2.new(0,48,0,22) actBtn.Position=UDim2.new(1,-54,0.5,-11)
    actBtn.BackgroundColor3=C_GREEN actBtn.BorderSizePixel=0
    actBtn.Text="USE" actBtn.TextColor3=C_BG actBtn.Font=Enum.Font.GothamBold actBtn.TextSize=9 actBtn.ZIndex=6
    Instance.new("UICorner",actBtn).CornerRadius=UDim.new(0,7)
    setTPDownVisual=function(state)
        TweenService:Create(actBtn,TweenInfo.new(0.15),{BackgroundColor3=state and C_WHITE or C_GREEN}):Play()
    end
    VisualSetters.TPDown=setTPDownVisual
    actBtn.MouseButton1Click:Connect(function()
        playTick()
        if setTPDownVisual then setTPDownVisual(true) end
        task.spawn(runTPDown)
    end)
    actBtn.MouseEnter:Connect(function() TweenService:Create(cStroke,TweenInfo.new(0.15),{Transparency=0.3}):Play() end)
    actBtn.MouseLeave:Connect(function() TweenService:Create(cStroke,TweenInfo.new(0.15),{Transparency=0.8}):Play() end)
end

-- ── MOVEMENT TAB ──────────────────────────────────────────────
order=0
createSection(movementPage,"Auto Play",O())
do
    local row=Instance.new("Frame",movementPage)
    row.Size=UDim2.new(0.96,0,0,32) row.BackgroundColor3=C_CARD
    row.BackgroundTransparency=0.3 row.BorderSizePixel=0
    row.LayoutOrder=O() row.ZIndex=7
    Instance.new("UICorner",row).CornerRadius=UDim.new(0,8)
    local rowStroke=Instance.new("UIStroke",row)
    rowStroke.Color=C_DIM rowStroke.Thickness=1 rowStroke.Transparency=0.6

    local lbl=Instance.new("TextLabel",row)
    lbl.Size=UDim2.new(1,0,1,0) lbl.Position=UDim2.new(0,0,0,0)
    lbl.BackgroundTransparency=1 lbl.Text="Auto Play"
    lbl.TextColor3=Color3.new(1,1,1) lbl.Font=Enum.Font.GothamBold lbl.TextSize=11
    lbl.TextXAlignment=Enum.TextXAlignment.Center lbl.ZIndex=8

    local btn=Instance.new("TextButton",row)
    btn.Size=UDim2.new(1,0,1,0) btn.BackgroundTransparency=1 btn.Text="" btn.ZIndex=9

    local picker=Instance.new("Frame",gui)
    picker.Name="SoloAutoPlayPicker"
    picker.Size=UDim2.new(0,140,0,70)
    picker.BackgroundColor3=C_CARD
    picker.BorderSizePixel=0 picker.ZIndex=50 picker.Visible=false
    Instance.new("UICorner",picker).CornerRadius=UDim.new(0,8)
    local pickerStroke=Instance.new("UIStroke",picker)
    pickerStroke.Color=C_GREEN pickerStroke.Thickness=1 pickerStroke.Transparency=0.4

    local pickerLbl=Instance.new("TextLabel",picker)
    pickerLbl.Size=UDim2.new(1,0,0,20) pickerLbl.Position=UDim2.new(0,0,0,4)
    pickerLbl.BackgroundTransparency=1 pickerLbl.Text="Choose Side"
    pickerLbl.TextColor3=C_DIM pickerLbl.Font=Enum.Font.GothamBold pickerLbl.TextSize=9
    pickerLbl.TextXAlignment=Enum.TextXAlignment.Center pickerLbl.ZIndex=51

    local function makePickerBtn(txt,xPos)
        local b=Instance.new("TextButton",picker)
        b.Size=UDim2.new(0,58,0,28) b.Position=UDim2.new(0,xPos,0,28)
        b.BackgroundColor3=C_CARD b.BorderSizePixel=0
        b.Text=txt b.TextColor3=Color3.new(1,1,1)
        b.Font=Enum.Font.GothamBold b.TextSize=11 b.ZIndex=52
        b.AutoButtonColor=false
        Instance.new("UICorner",b).CornerRadius=UDim.new(0,8)
        local bs=Instance.new("UIStroke",b)
        bs.Color=C_GREEN bs.Thickness=1 bs.Transparency=0.6
        b.MouseEnter:Connect(function() TweenService:Create(b,TweenInfo.new(0.1),{BackgroundColor3=C_GREEN}):Play() end)
        b.MouseLeave:Connect(function() TweenService:Create(b,TweenInfo.new(0.1),{BackgroundColor3=C_CARD}):Play() end)
        return b
    end
    local btnLeft=makePickerBtn("Left",6)
    local btnRight=makePickerBtn("Right",76)

    local function showPicker()
        local abs=row.AbsolutePosition
        local sz=row.AbsoluteSize
        picker.Position=UDim2.new(0,abs.X,0,abs.Y+sz.Y+4)
        picker.Visible=true
    end
    local function hidePicker() picker.Visible=false end

    local apIsOn=false
    local apChosenSide=nil

    local function apSetVisual(state)
        apIsOn=state
        TweenService:Create(rowStroke,TweenInfo.new(0.2),{Color=state and C_GREEN or C_DIM,Transparency=state and 0 or 0.6}):Play()
        TweenService:Create(lbl,TweenInfo.new(0.15),{TextColor3=state and C_GREEN or Color3.new(1,1,1)}):Play()
    end

    local function apMenuStop()
        apIsOn=false
        if aplOn then aplOn=false if aplConn then aplConn:Disconnect() aplConn=nil end aplPhase=1
            local ch=player.Character if ch then local h=ch:FindFirstChildOfClass("Humanoid") if h then h:Move(Vector3.zero) end end
        end
        if aprOn then aprOn=false if aprConn then aprConn:Disconnect() aprConn=nil end aprPhase=1
            local ch=player.Character if ch then local h=ch:FindFirstChildOfClass("Humanoid") if h then h:Move(Vector3.zero) end end
        end
        apSetVisual(false)
    end

    local function apMenuStart(side)
        apChosenSide=side apIsOn=true
        if side=="LEFT" then
            if aprOn then aprOn=false if aprConn then aprConn:Disconnect() aprConn=nil end aprPhase=1 end
            aplOn=true startAutoPlayLeft()
        else
            if aplOn then aplOn=false if aplConn then aplConn:Disconnect() aplConn=nil end aplPhase=1 end
            aprOn=true startAutoPlayRight()
        end
        apSetVisual(true)
    end

    btnLeft.MouseButton1Click:Connect(function()
        hidePicker()
        if apIsOn then apMenuStop() end
        apMenuStart("LEFT") playTick()
    end)
    btnRight.MouseButton1Click:Connect(function()
        hidePicker()
        if apIsOn then apMenuStop() end
        apMenuStart("RIGHT") playTick()
    end)

    UserInputService.InputBegan:Connect(function(inp,gpe)
        if not picker.Visible then return end
        if inp.UserInputType==Enum.UserInputType.MouseButton1 then
            local pos=inp.Position
            local pa=picker.AbsolutePosition; local ps=picker.AbsoluteSize
            if pos.X<pa.X or pos.X>pa.X+ps.X or pos.Y<pa.Y or pos.Y>pa.Y+ps.Y then
                hidePicker()
            end
        end
    end)

    local holdThread=nil
    local holdFired=false

    btn.MouseButton1Down:Connect(function()
        holdFired=false
        holdThread=task.delay(3,function()
            holdFired=true
            showPicker()
        end)
    end)
    btn.MouseButton1Up:Connect(function()
        if holdThread then task.cancel(holdThread) holdThread=nil end
        if holdFired then return end
        if apIsOn then
            apMenuStop()
        else
            if apChosenSide then
                apMenuStart(apChosenSide)
            else
                showPicker()
            end
        end
        playTick()
    end)

    btn.MouseEnter:Connect(function() TweenService:Create(row,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(8,30,8),BackgroundTransparency=0}):Play() end)
    btn.MouseLeave:Connect(function() TweenService:Create(row,TweenInfo.new(0.15),{BackgroundColor3=C_CARD,BackgroundTransparency=0.3}):Play() end)

    player.CharacterAdded:Connect(function()
        aplOn=false aprOn=false apIsOn=false apSetVisual(false)
        if apChosenSide then
            task.wait(1.2)
            if apChosenSide then apMenuStart(apChosenSide) end
        end
    end)
    player.CharacterRemoving:Connect(function()
        aplOn=false aprOn=false apIsOn=false apSetVisual(false)
    end)
end
createSection(movementPage,"Speed",O())
do
    local function makeSpeedInput(lbl,getter,setter,ord)
        local c=Instance.new("Frame",movementPage)
        c.Size=UDim2.new(1,0,0,30) c.BackgroundColor3=C_CARD c.BorderSizePixel=0 c.LayoutOrder=ord c.ZIndex=3
        Instance.new("UICorner",c).CornerRadius=UDim.new(0,7)
        local stroke=Instance.new("UIStroke",c) stroke.Color=C_GREEN stroke.Thickness=1 stroke.Transparency=0.8
        local label=Instance.new("TextLabel",c)
        label.Size=UDim2.new(0.6,0,1,0) label.Position=UDim2.new(0,8,0,0)
        label.BackgroundTransparency=1 label.Text=lbl label.TextColor3=C_WHITE
        label.Font=Enum.Font.GothamBold label.TextSize=10 label.TextXAlignment=Enum.TextXAlignment.Left label.ZIndex=4
        local input=Instance.new("TextBox",c)
        input.Size=UDim2.new(0,55,0,20) input.Position=UDim2.new(1,-62,0.5,-10)
        input.BackgroundColor3=C_BG input.BorderSizePixel=0 input.Text=tostring(getter())
        input.TextColor3=C_WHITE input.Font=Enum.Font.GothamBold input.TextSize=10 input.ZIndex=4
        Instance.new("UICorner",input).CornerRadius=UDim.new(0,5)
        local iStroke=Instance.new("UIStroke",input) iStroke.Color=C_GREEN iStroke.Thickness=1 iStroke.Transparency=0.5
        input.FocusLost:Connect(function()
            local n=tonumber(input.Text)
            if n then setter(math.clamp(n,1,9999)) input.Text=tostring(n)
            else input.Text=tostring(getter()) end
        end)
    end
    makeSpeedInput("Normal Speed",function() return NORMAL_SPEED end,function(v) NORMAL_SPEED=v saveSettings() end,O())
    makeSpeedInput("Carry Speed",function() return SLOW_SPEED end,function(v) SLOW_SPEED=v saveSettings() end,O())
    makeSpeedInput("Lag Speed",function() return LAGGER_SPEED end,function(v) LAGGER_SPEED=v end,O())
end

-- ── AUTO TAB ──────────────────────────────────────────────────
order=0
createSection(autoPage,"Auto TP",O())
createToggle(autoPage,"Auto TP","AutoTP",false,O(),function(on)
    G_tpAutoEnabled=on
    if on then task.spawn(function() refreshMyPlotSide() end) end
end)
createSection(autoPage,"Auto Steal",O())
createToggle(autoPage,"Auto Steal","AutoSteal",true,O(),function(on)
    autoStealEnabled=true
    startAutoSteal()
end)
createSection(autoPage,"Steal Settings",O())
do
    local function makeInput(lbl,getter,setter,ord)
        local c=Instance.new("Frame",autoPage)
        c.Size=UDim2.new(1,0,0,30) c.BackgroundColor3=C_CARD c.BorderSizePixel=0 c.LayoutOrder=ord c.ZIndex=3
        Instance.new("UICorner",c).CornerRadius=UDim.new(0,7)
        local stroke=Instance.new("UIStroke",c) stroke.Color=C_GREEN stroke.Thickness=1 stroke.Transparency=0.8
        local label=Instance.new("TextLabel",c)
        label.Size=UDim2.new(0.6,0,1,0) label.Position=UDim2.new(0,8,0,0)
        label.BackgroundTransparency=1 label.Text=lbl label.TextColor3=C_WHITE
        label.Font=Enum.Font.GothamBold label.TextSize=10 label.TextXAlignment=Enum.TextXAlignment.Left label.ZIndex=4
        local input=Instance.new("TextBox",c)
        input.Size=UDim2.new(0,55,0,20) input.Position=UDim2.new(1,-62,0.5,-10)
        input.BackgroundColor3=C_BG input.BorderSizePixel=0 input.Text=tostring(getter())
        input.TextColor3=C_WHITE input.Font=Enum.Font.GothamBold input.TextSize=10 input.ZIndex=4
        Instance.new("UICorner",input).CornerRadius=UDim.new(0,5)
        local iStroke=Instance.new("UIStroke",input) iStroke.Color=C_GREEN iStroke.Thickness=1 iStroke.Transparency=0.5
        input.FocusLost:Connect(function()
            local n=tonumber(input.Text)
            if n then setter(n) input.Text=tostring(n) else input.Text=tostring(getter()) end
        end)
        return input
    end
    makeInput("Steal Radius",function() return STEAL_RADIUS end,function(v) STEAL_RADIUS=math.clamp(v,5,200) end,O())
    makeInput("Steal Duration",function() return STEAL_DURATION end,function(v) STEAL_DURATION=math.max(0.05,v) end,O())
end

-- ── VISUAL TAB ────────────────────────────────────────────────
order=0
createSection(visualPage,"Display",O())

-- GUI Scale changer
do
    local container=Instance.new("Frame",visualPage)
    container.Size=UDim2.new(1,0,0,42) container.BackgroundColor3=C_CARD
    container.BorderSizePixel=0 container.LayoutOrder=O() container.ZIndex=3
    Instance.new("UICorner",container).CornerRadius=UDim.new(0,8)
    local cStroke=Instance.new("UIStroke",container)
    cStroke.Color=C_GREEN cStroke.Thickness=1 cStroke.Transparency=0.8
    local label=Instance.new("TextLabel",container)
    label.Size=UDim2.new(0,90,1,0) label.Position=UDim2.new(0,10,0,0)
    label.BackgroundTransparency=1 label.Text="GUI Scale" label.TextColor3=C_WHITE
    label.Font=Enum.Font.GothamBold label.TextSize=13 label.TextXAlignment=Enum.TextXAlignment.Left label.ZIndex=4
    local leftArrow=Instance.new("TextButton",container)
    leftArrow.Size=UDim2.new(0,26,0,26) leftArrow.Position=UDim2.new(1,-120,0.5,-13)
    leftArrow.BackgroundColor3=Color3.fromRGB(30, 30, 30) leftArrow.BorderSizePixel=0
    leftArrow.Text="◀" leftArrow.TextColor3=C_GREEN leftArrow.Font=Enum.Font.GothamBold leftArrow.TextSize=12 leftArrow.ZIndex=5
    Instance.new("UICorner",leftArrow).CornerRadius=UDim.new(0,6)
    local laStroke=Instance.new("UIStroke",leftArrow) laStroke.Color=C_GREEN laStroke.Thickness=1 laStroke.Transparency=0.5
    local valueBox=Instance.new("TextBox",container)
    valueBox.Size=UDim2.new(0,50,0,26) valueBox.Position=UDim2.new(1,-90,0.5,-13)
    valueBox.BackgroundColor3=C_BG valueBox.BorderSizePixel=0
    valueBox.Text=tostring(math.floor(UI_SCALE*100+0.5)) valueBox.TextColor3=C_WHITE
    valueBox.Font=Enum.Font.GothamBlack valueBox.TextSize=13 valueBox.TextXAlignment=Enum.TextXAlignment.Center valueBox.ZIndex=5
    Instance.new("UICorner",valueBox).CornerRadius=UDim.new(0,6)
    local vbStroke=Instance.new("UIStroke",valueBox) vbStroke.Color=C_GREEN vbStroke.Thickness=1 vbStroke.Transparency=0.4
    local rightArrow=Instance.new("TextButton",container)
    rightArrow.Size=UDim2.new(0,26,0,26) rightArrow.Position=UDim2.new(1,-36,0.5,-13)
    rightArrow.BackgroundColor3=Color3.fromRGB(30, 30, 30) rightArrow.BorderSizePixel=0
    rightArrow.Text="▶" rightArrow.TextColor3=C_GREEN rightArrow.Font=Enum.Font.GothamBold rightArrow.TextSize=12 rightArrow.ZIndex=5
    Instance.new("UICorner",rightArrow).CornerRadius=UDim.new(0,6)
    local raStroke=Instance.new("UIStroke",rightArrow) raStroke.Color=C_GREEN raStroke.Thickness=1 raStroke.Transparency=0.5
    local function getOrCreateScale(guiFrame)
        if not guiFrame then return nil end
        local existing=guiFrame:FindFirstChildOfClass("UIScale")
        if existing then return existing end
        local s=Instance.new("UIScale") s.Scale=UI_SCALE s.Parent=guiFrame return s
    end
    local function applyScaleToAll(pct)
        pct=math.clamp(pct,50,100) UI_SCALE=pct/100 valueBox.Text=tostring(pct)
        TweenService:Create(getOrCreateScale(main),TweenInfo.new(0.18,Enum.EasingStyle.Quint),{Scale=UI_SCALE}):Play()
        if mobileButtonContainer then TweenService:Create(getOrCreateScale(mobileButtonContainer),TweenInfo.new(0.18,Enum.EasingStyle.Quint),{Scale=UI_SCALE}):Play() end
        if progressBar then TweenService:Create(getOrCreateScale(progressBar),TweenInfo.new(0.18,Enum.EasingStyle.Quint),{Scale=UI_SCALE}):Play() end
        TweenService:Create(vbStroke,TweenInfo.new(0.1),{Transparency=0}):Play()
        task.delay(0.3,function() TweenService:Create(vbStroke,TweenInfo.new(0.2),{Transparency=0.4}):Play() end)
    end
    local function getCurrentPct() return math.clamp(math.floor(UI_SCALE*100+0.5),50,100) end
    leftArrow.MouseButton1Click:Connect(function() playTick() applyScaleToAll(math.max(50,getCurrentPct()-5)) end)
    rightArrow.MouseButton1Click:Connect(function() playTick() applyScaleToAll(math.min(100,getCurrentPct()+5)) end)
    valueBox.FocusLost:Connect(function()
        local n=tonumber(valueBox.Text)
        if n then applyScaleToAll(math.clamp(math.floor(n+0.5),50,100)) else valueBox.Text=tostring(getCurrentPct()) end
    end)
    leftArrow.MouseEnter:Connect(function() TweenService:Create(laStroke,TweenInfo.new(0.12),{Transparency=0.1}):Play() end)
    leftArrow.MouseLeave:Connect(function() TweenService:Create(laStroke,TweenInfo.new(0.12),{Transparency=0.5}):Play() end)
    rightArrow.MouseEnter:Connect(function() TweenService:Create(raStroke,TweenInfo.new(0.12),{Transparency=0.1}):Play() end)
    rightArrow.MouseLeave:Connect(function() TweenService:Create(raStroke,TweenInfo.new(0.12),{Transparency=0.5}):Play() end)
end

-- Button Size slider
createSection(visualPage,"Button Size",O())
do
    local BTN_SCALE = 1.0
    local container=Instance.new("Frame",visualPage)
    container.Size=UDim2.new(1,0,0,42) container.BackgroundColor3=C_CARD
    container.BorderSizePixel=0 container.LayoutOrder=O() container.ZIndex=3
    Instance.new("UICorner",container).CornerRadius=UDim.new(0,14)
    local cStroke=Instance.new("UIStroke",container) cStroke.Color=C_GREEN cStroke.Thickness=1 cStroke.Transparency=0.8
    local lbl=Instance.new("TextLabel",container)
    lbl.Size=UDim2.new(0,90,1,0) lbl.Position=UDim2.new(0,10,0,0)
    lbl.BackgroundTransparency=1 lbl.Text="Button Size" lbl.TextColor3=C_WHITE
    lbl.Font=Enum.Font.GothamBold lbl.TextSize=11 lbl.TextXAlignment=Enum.TextXAlignment.Left lbl.ZIndex=5
    local leftArrow=Instance.new("TextButton",container)
    leftArrow.Size=UDim2.new(0,26,0,26) leftArrow.Position=UDim2.new(1,-120,0.5,-13)
    leftArrow.BackgroundColor3=Color3.fromRGB(30,30,30) leftArrow.BorderSizePixel=0
    leftArrow.Text="◀" leftArrow.TextColor3=C_GREEN leftArrow.Font=Enum.Font.GothamBold leftArrow.TextSize=12 leftArrow.ZIndex=5
    Instance.new("UICorner",leftArrow).CornerRadius=UDim.new(0,6)
    local laStroke=Instance.new("UIStroke",leftArrow) laStroke.Color=C_GREEN laStroke.Thickness=1 laStroke.Transparency=0.5
    local valueBox=Instance.new("TextBox",container)
    valueBox.Size=UDim2.new(0,50,0,26) valueBox.Position=UDim2.new(1,-90,0.5,-13)
    valueBox.BackgroundColor3=C_BG valueBox.BorderSizePixel=0
    valueBox.Text="100" valueBox.TextColor3=C_WHITE
    valueBox.Font=Enum.Font.GothamBlack valueBox.TextSize=13 valueBox.TextXAlignment=Enum.TextXAlignment.Center valueBox.ZIndex=5
    Instance.new("UICorner",valueBox).CornerRadius=UDim.new(0,6)
    local vbStroke=Instance.new("UIStroke",valueBox) vbStroke.Color=C_GREEN vbStroke.Thickness=1 vbStroke.Transparency=0.4
    local rightArrow=Instance.new("TextButton",container)
    rightArrow.Size=UDim2.new(0,26,0,26) rightArrow.Position=UDim2.new(1,-36,0.5,-13)
    rightArrow.BackgroundColor3=Color3.fromRGB(30,30,30) rightArrow.BorderSizePixel=0
    rightArrow.Text="▶" rightArrow.TextColor3=C_GREEN rightArrow.Font=Enum.Font.GothamBold rightArrow.TextSize=12 rightArrow.ZIndex=5
    Instance.new("UICorner",rightArrow).CornerRadius=UDim.new(0,6)
    local raStroke=Instance.new("UIStroke",rightArrow) raStroke.Color=C_GREEN raStroke.Thickness=1 raStroke.Transparency=0.5
    local function applyBtnScale(pct)
        pct = math.clamp(pct, 50, 150)
        BTN_SCALE = pct / 100
        valueBox.Text = tostring(pct)
        for _, child in ipairs(gui:GetChildren()) do
            if child:IsA("TextButton") and child.ZIndex >= 100 then
                local sc = child:FindFirstChildOfClass("UIScale")
                if not sc then sc = Instance.new("UIScale", child) end
                TweenService:Create(sc, TweenInfo.new(0.18, Enum.EasingStyle.Quint), {Scale = BTN_SCALE}):Play()
            end
        end
        TweenService:Create(vbStroke,TweenInfo.new(0.1),{Transparency=0}):Play()
        task.delay(0.3,function() TweenService:Create(vbStroke,TweenInfo.new(0.2),{Transparency=0.4}):Play() end)
    end
    local function getCurPct() return math.clamp(math.floor(BTN_SCALE*100+0.5),50,150) end
    leftArrow.MouseButton1Click:Connect(function() playTick() applyBtnScale(math.max(50,getCurPct()-5)) end)
    rightArrow.MouseButton1Click:Connect(function() playTick() applyBtnScale(math.min(150,getCurPct()+5)) end)
    valueBox.FocusLost:Connect(function()
        local n=tonumber(valueBox.Text)
        if n then applyBtnScale(math.clamp(math.floor(n+0.5),50,150)) else valueBox.Text=tostring(getCurPct()) end
    end)
    leftArrow.MouseEnter:Connect(function() TweenService:Create(laStroke,TweenInfo.new(0.12),{Transparency=0.1}):Play() end)
    leftArrow.MouseLeave:Connect(function() TweenService:Create(laStroke,TweenInfo.new(0.12),{Transparency=0.5}):Play() end)
    rightArrow.MouseEnter:Connect(function() TweenService:Create(raStroke,TweenInfo.new(0.12),{Transparency=0.1}):Play() end)
    rightArrow.MouseLeave:Connect(function() TweenService:Create(raStroke,TweenInfo.new(0.12),{Transparency=0.5}):Play() end)
end

-- Lock Side Panel toggle
createSection(visualPage,"Settings",O())
do
    local cont=Instance.new("Frame",visualPage)
    cont.Size=UDim2.new(1,0,0,42) cont.BackgroundColor3=C_CARD
    cont.BorderSizePixel=0 cont.LayoutOrder=O() cont.ZIndex=3
    Instance.new("UICorner",cont).CornerRadius=UDim.new(0,14)
    local cStroke=Instance.new("UIStroke",cont) cStroke.Color=C_GREEN cStroke.Thickness=1 cStroke.Transparency=0.8
    local icon=Instance.new("TextLabel",cont)
    icon.Size=UDim2.new(0,22,1,0) icon.Position=UDim2.new(0,8,0,0)
    icon.BackgroundTransparency=1 icon.Text="🔒" icon.TextColor3=C_WHITE
    icon.Font=Enum.Font.GothamBold icon.TextSize=14 icon.TextXAlignment=Enum.TextXAlignment.Left icon.ZIndex=5
    local lbl=Instance.new("TextLabel",cont)
    lbl.Size=UDim2.new(1,-100,1,0) lbl.Position=UDim2.new(0,34,0,0)
    lbl.BackgroundTransparency=1 lbl.Text="Unlock Buttons" lbl.TextColor3=C_WHITE
    lbl.Font=Enum.Font.GothamBold lbl.TextSize=11 lbl.TextXAlignment=Enum.TextXAlignment.Left lbl.ZIndex=5
    local swBg=Instance.new("Frame",cont)
    swBg.Size=UDim2.new(0,38,0,18) swBg.Position=UDim2.new(1,-46,0.5,-9)
    swBg.BackgroundColor3=C_SW_OFF swBg.BorderSizePixel=0 swBg.ZIndex=5
    Instance.new("UICorner",swBg).CornerRadius=UDim.new(1,0)
    local swC=Instance.new("Frame",swBg)
    swC.Size=UDim2.new(0,14,0,14) swC.Position=UDim2.new(0,2,0.5,-7)
    swC.BackgroundColor3=C_WHITE swC.BorderSizePixel=0 swC.ZIndex=6
    Instance.new("UICorner",swC).CornerRadius=UDim.new(1,0)
    local _unlockOn=false
    local btn=Instance.new("TextButton",cont)
    btn.Size=UDim2.new(1,0,1,0) btn.BackgroundTransparency=1 btn.Text="" btn.ZIndex=7
    btn.MouseButton1Click:Connect(function()
        playTick()
        _unlockOn=not _unlockOn
        _sidePanelLocked=not _unlockOn
        icon.Text=_unlockOn and "🔓" or "🔒"
        TweenService:Create(swBg,TweenInfo.new(0.2,Enum.EasingStyle.Quint),{BackgroundColor3=_unlockOn and C_GREEN or C_SW_OFF}):Play()
        TweenService:Create(swC,TweenInfo.new(0.2,Enum.EasingStyle.Back),{Position=_unlockOn and UDim2.new(1,-16,0.5,-7) or UDim2.new(0,2,0.5,-7)}):Play()
        TweenService:Create(cont,TweenInfo.new(0.15),{BackgroundColor3=_unlockOn and Color3.fromRGB(28, 28, 28) or C_CARD}):Play()
    end)
    btn.MouseEnter:Connect(function() TweenService:Create(cStroke,TweenInfo.new(0.15),{Transparency=0.3}):Play() TweenService:Create(cont,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(32, 32, 32)}):Play() end)
    btn.MouseLeave:Connect(function() TweenService:Create(cStroke,TweenInfo.new(0.15),{Transparency=0.8}):Play() if not _sidePanelLocked then TweenService:Create(cont,TweenInfo.new(0.15),{BackgroundColor3=C_CARD}):Play() end end)
end

-- Op Animation toggle
createSection(visualPage,"Op Animation",O())
do
    local Anims = {
        idle1    = "rbxassetid://133806214992291",
        idle2    = "rbxassetid://94970088341563",
        walk     = "rbxassetid://707897309",
        run      = "rbxassetid://707861613",
        jump     = "rbxassetid://116936326516985",
        fall     = "rbxassetid://116936326516985",
        climb    = "rbxassetid://116936326516985",
        swim     = "rbxassetid://116936326516985",
        swimidle = "rbxassetid://116936326516985",
    }

    local goodAnimEnabled = false
    local goodAnimConn    = nil
    local origAnims       = nil

    local function saveOrigAnims(char)
        local a = char:FindFirstChild("Animate"); if not a then return end
        local function g(o) return o and o.AnimationId or nil end
        origAnims = {
            idle1    = g(a.idle    and a.idle.Animation1),
            idle2    = g(a.idle    and a.idle.Animation2),
            walk     = g(a.walk    and a.walk.WalkAnim),
            run      = g(a.run     and a.run.RunAnim),
            jump     = g(a.jump    and a.jump.JumpAnim),
            fall     = g(a.fall    and a.fall.FallAnim),
            climb    = g(a.climb   and a.climb.ClimbAnim),
            swim     = g(a.swim    and a.swim.Swim),
            swimidle = g(a.swimidle and a.swimidle.SwimIdle),
        }
    end

    local function applyGoodAnims(char)
        local a = char:FindFirstChild("Animate"); if not a then return end
        local function s(o, id) if o then o.AnimationId = id end end
        s(a.idle    and a.idle.Animation1,    Anims.idle1)
        s(a.idle    and a.idle.Animation2,    Anims.idle2)
        s(a.walk    and a.walk.WalkAnim,      Anims.walk)
        s(a.run     and a.run.RunAnim,        Anims.run)
        s(a.jump    and a.jump.JumpAnim,      Anims.jump)
        s(a.fall    and a.fall.FallAnim,      Anims.fall)
        s(a.climb   and a.climb.ClimbAnim,    Anims.climb)
        s(a.swim    and a.swim.Swim,          Anims.swim)
        s(a.swimidle and a.swimidle.SwimIdle, Anims.swimidle)
    end

    local function restoreOrigAnims(char)
        if not origAnims then return end
        local a = char:FindFirstChild("Animate"); if not a then return end
        local function s(o, id) if o and id then o.AnimationId = id end end
        s(a.idle    and a.idle.Animation1,    origAnims.idle1)
        s(a.idle    and a.idle.Animation2,    origAnims.idle2)
        s(a.walk    and a.walk.WalkAnim,      origAnims.walk)
        s(a.run     and a.run.RunAnim,        origAnims.run)
        s(a.jump    and a.jump.JumpAnim,      origAnims.jump)
        s(a.fall    and a.fall.FallAnim,      origAnims.fall)
        s(a.climb   and a.climb.ClimbAnim,    origAnims.climb)
        s(a.swim    and a.swim.Swim,          origAnims.swim)
        s(a.swimidle and a.swimidle.SwimIdle, origAnims.swimidle)
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then for _, t in ipairs(hum:GetPlayingAnimationTracks()) do t:Stop(0) end end
    end

    local function startGoodAnim()
        local char = player.Character; if not char then return end
        saveOrigAnims(char)
        applyGoodAnims(char)
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then for _, t in ipairs(hum:GetPlayingAnimationTracks()) do t:Stop(0) end end
        if goodAnimConn then goodAnimConn:Disconnect() end
        goodAnimConn = RunService.Heartbeat:Connect(function()
            if not goodAnimEnabled then return end
            local c = player.Character
            if c then applyGoodAnims(c) end
        end)
    end

    local function stopGoodAnim()
        if goodAnimConn then goodAnimConn:Disconnect(); goodAnimConn = nil end
        local char = player.Character
        if char then restoreOrigAnims(char) end
    end

    player.CharacterAdded:Connect(function(c)
        if not goodAnimEnabled then return end
        task.wait(0.8)
        saveOrigAnims(c)
        applyGoodAnims(c)
        local hum = c:FindFirstChildOfClass("Humanoid")
        if hum then for _, t in ipairs(hum:GetPlayingAnimationTracks()) do t:Stop(0) end end
    end)

    local cont = Instance.new("Frame", visualPage)
    cont.Size = UDim2.new(1,0,0,42) cont.BackgroundColor3 = C_CARD
    cont.BorderSizePixel = 0 cont.LayoutOrder = O() cont.ZIndex = 3
    Instance.new("UICorner", cont).CornerRadius = UDim.new(0,14)
    local cStroke = Instance.new("UIStroke", cont)
    cStroke.Color = C_GREEN cStroke.Thickness = 1 cStroke.Transparency = 0.8
    local lbl = Instance.new("TextLabel", cont)
    lbl.Size = UDim2.new(1,-80,1,0) lbl.Position = UDim2.new(0,10,0,0)
    lbl.BackgroundTransparency = 1 lbl.Text = "Op Animation" lbl.TextColor3 = C_WHITE
    lbl.Font = Enum.Font.GothamBold lbl.TextSize = 11
    lbl.TextXAlignment = Enum.TextXAlignment.Left lbl.ZIndex = 5
    local swBg = Instance.new("Frame", cont)
    swBg.Size = UDim2.new(0,38,0,18) swBg.Position = UDim2.new(1,-46,0.5,-9)
    swBg.BackgroundColor3 = C_SW_OFF swBg.BorderSizePixel = 0 swBg.ZIndex = 5
    Instance.new("UICorner", swBg).CornerRadius = UDim.new(1,0)
    local swC = Instance.new("Frame", swBg)
    swC.Size = UDim2.new(0,14,0,14) swC.Position = UDim2.new(0,2,0.5,-7)
    swC.BackgroundColor3 = C_WHITE swC.BorderSizePixel = 0 swC.ZIndex = 6
    Instance.new("UICorner", swC).CornerRadius = UDim.new(1,0)
    local animBtn = Instance.new("TextButton", cont)
    animBtn.Size = UDim2.new(1,0,1,0) animBtn.BackgroundTransparency = 1
    animBtn.Text = "" animBtn.ZIndex = 7
    animBtn.MouseButton1Click:Connect(function()
        playTick()
        goodAnimEnabled = not goodAnimEnabled
        TweenService:Create(swBg,TweenInfo.new(0.2,Enum.EasingStyle.Quint),{BackgroundColor3=goodAnimEnabled and C_GREEN or C_SW_OFF}):Play()
        TweenService:Create(swC,TweenInfo.new(0.2,Enum.EasingStyle.Back),{Position=goodAnimEnabled and UDim2.new(1,-16,0.5,-7) or UDim2.new(0,2,0.5,-7)}):Play()
        if goodAnimEnabled then startGoodAnim() else stopGoodAnim() end
    end)
    animBtn.MouseEnter:Connect(function()
        TweenService:Create(cStroke,TweenInfo.new(0.15),{Transparency=0.3}):Play()
        TweenService:Create(cont,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(35,35,35)}):Play()
    end)
    animBtn.MouseLeave:Connect(function()
        TweenService:Create(cStroke,TweenInfo.new(0.15),{Transparency=0.8}):Play()
        TweenService:Create(cont,TweenInfo.new(0.15),{BackgroundColor3=C_CARD}):Play()
    end)
end

-- Progress bar widget
local progressBar=Instance.new("Frame",gui)
progressBar.Size=UDim2.new(0,220,0,78) progressBar.Position=UDim2.new(0.5,-110,0,10)
_registerBtn("progressBar", progressBar)
progressBar.BackgroundColor3=Color3.fromRGB(10, 10, 10) progressBar.BorderSizePixel=0
progressBar.Active=true progressBar.ZIndex=50
Instance.new("UICorner",progressBar).CornerRadius=UDim.new(0,16)
local pbStroke=Instance.new("UIStroke",progressBar) pbStroke.Color=C_GREEN pbStroke.Thickness=1.5 pbStroke.Transparency=0.4
do local sc=Instance.new("UIScale",progressBar) sc.Scale=UI_SCALE end
local hubTitleLbl=Instance.new("TextLabel",progressBar)
hubTitleLbl.Size=UDim2.new(1,0,0,22) hubTitleLbl.Position=UDim2.new(0,0,0,4)
hubTitleLbl.BackgroundTransparency=1 hubTitleLbl.Text="SOLO DUELS"
hubTitleLbl.TextColor3=C_GREEN hubTitleLbl.Font=Enum.Font.GothamBlack
hubTitleLbl.TextSize=15 hubTitleLbl.TextXAlignment=Enum.TextXAlignment.Center hubTitleLbl.ZIndex=55
local hubSep=Instance.new("Frame",progressBar)
hubSep.Size=UDim2.new(0.85,0,0,1) hubSep.Position=UDim2.new(0.075,0,0,28)
hubSep.BackgroundColor3=C_GREEN hubSep.BackgroundTransparency=0.5 hubSep.BorderSizePixel=0 hubSep.ZIndex=54
do
    local drag,ds,sp=false
    progressBar.InputBegan:Connect(function(inp)
        if inp.UserInputType==Enum.UserInputType.MouseButton1 or inp.UserInputType==Enum.UserInputType.Touch then
            drag=true ds=inp.Position sp=progressBar.Position
            inp.Changed:Connect(function() if inp.UserInputState==Enum.UserInputState.End then drag=false end end)
        end
    end)
    UserInputService.InputChanged:Connect(function(inp)
        if drag and (inp.UserInputType==Enum.UserInputType.MouseMovement or inp.UserInputType==Enum.UserInputType.Touch) then
            local d=inp.Position-ds progressBar.Position=UDim2.new(sp.X.Scale,sp.X.Offset+d.X,sp.Y.Scale,sp.Y.Offset+d.Y)
        end
    end)
end
ProgressLabel=Instance.new("TextLabel",progressBar)
ProgressLabel.Size=UDim2.new(0.55,0,0,14) ProgressLabel.Position=UDim2.new(0,10,0,28)
ProgressLabel.BackgroundTransparency=1 ProgressLabel.Text="READY"
ProgressLabel.TextColor3=C_GREEN ProgressLabel.Font=Enum.Font.GothamBlack
ProgressLabel.TextSize=9 ProgressLabel.TextXAlignment=Enum.TextXAlignment.Left ProgressLabel.ZIndex=51
ProgressPctLabel=Instance.new("TextLabel",progressBar)
ProgressPctLabel.Size=UDim2.new(0.45,-8,0,14) ProgressPctLabel.Position=UDim2.new(0.55,0,0,28)
ProgressPctLabel.BackgroundTransparency=1 ProgressPctLabel.Text=""
ProgressPctLabel.TextColor3=C_WHITE ProgressPctLabel.Font=Enum.Font.GothamBlack
ProgressPctLabel.TextSize=9 ProgressPctLabel.TextXAlignment=Enum.TextXAlignment.Right ProgressPctLabel.ZIndex=51
local pTrack=Instance.new("Frame",progressBar)
pTrack.Size=UDim2.new(1,-20,0,4) pTrack.Position=UDim2.new(0,10,0,46)
pTrack.BackgroundColor3=Color3.fromRGB(30, 30, 30) pTrack.BorderSizePixel=0 pTrack.ZIndex=50
Instance.new("UICorner",pTrack).CornerRadius=UDim.new(1,0)
local pTrackStroke=Instance.new("UIStroke",pTrack) pTrackStroke.Color=C_GREEN pTrackStroke.Thickness=1 pTrackStroke.Transparency=0.85
ProgressBarFill=Instance.new("Frame",pTrack)
ProgressBarFill.Size=UDim2.new(0,0,1,0) ProgressBarFill.BackgroundColor3=C_GREEN
ProgressBarFill.BorderSizePixel=0 ProgressBarFill.ZIndex=51
Instance.new("UICorner",ProgressBarFill).CornerRadius=UDim.new(1,0)
local radLbl=Instance.new("TextLabel",progressBar)
radLbl.Size=UDim2.new(0,38,0,18) radLbl.Position=UDim2.new(0,10,0,54)
radLbl.BackgroundTransparency=1 radLbl.Text="RADIUS"
radLbl.TextColor3=C_DIM radLbl.Font=Enum.Font.GothamBold radLbl.TextSize=8
radLbl.TextXAlignment=Enum.TextXAlignment.Left radLbl.ZIndex=51
local radMinus=Instance.new("TextButton",progressBar)
radMinus.Size=UDim2.new(0,20,0,18) radMinus.Position=UDim2.new(0,52,0,54)
radMinus.BackgroundColor3=Color3.fromRGB(28, 28, 28) radMinus.BorderSizePixel=0
radMinus.Text="−" radMinus.TextColor3=C_GREEN
radMinus.Font=Enum.Font.GothamBlack radMinus.TextSize=13 radMinus.ZIndex=52
Instance.new("UICorner",radMinus).CornerRadius=UDim.new(0,5)
RadiusInput=Instance.new("TextBox",progressBar)
RadiusInput.Size=UDim2.new(0,60,0,18) RadiusInput.Position=UDim2.new(0,75,0,54)
RadiusInput.BackgroundColor3=Color3.fromRGB(22, 22, 22) RadiusInput.BorderSizePixel=0
RadiusInput.Text=tostring(STEAL_RADIUS) RadiusInput.TextColor3=C_WHITE
RadiusInput.Font=Enum.Font.GothamBlack RadiusInput.TextSize=10
RadiusInput.TextXAlignment=Enum.TextXAlignment.Center
RadiusInput.ClearTextOnFocus=false RadiusInput.ZIndex=52
Instance.new("UICorner",RadiusInput).CornerRadius=UDim.new(0,5)
local rInpStroke=Instance.new("UIStroke",RadiusInput) rInpStroke.Color=C_GREEN rInpStroke.Thickness=1 rInpStroke.Transparency=0.5
RadiusInput.FocusLost:Connect(function()
    local n=tonumber(RadiusInput.Text)
    if n then STEAL_RADIUS=math.clamp(math.floor(n),1,500) RadiusInput.Text=tostring(STEAL_RADIUS) end
end)
local radPlus=Instance.new("TextButton",progressBar)
radPlus.Size=UDim2.new(0,20,0,18) radPlus.Position=UDim2.new(0,138,0,54)
radPlus.BackgroundColor3=Color3.fromRGB(28, 28, 28) radPlus.BorderSizePixel=0
radPlus.Text="+" radPlus.TextColor3=C_GREEN
radPlus.Font=Enum.Font.GothamBlack radPlus.TextSize=13 radPlus.ZIndex=52
Instance.new("UICorner",radPlus).CornerRadius=UDim.new(0,5)
local radPlus5=Instance.new("TextButton",progressBar)
radPlus5.Size=UDim2.new(0,24,0,18) radPlus5.Position=UDim2.new(0,161,0,54)
radPlus5.BackgroundColor3=Color3.fromRGB(28, 28, 28) radPlus5.BorderSizePixel=0
radPlus5.Text="+5" radPlus5.TextColor3=C_GREEN
radPlus5.Font=Enum.Font.GothamBlack radPlus5.TextSize=8 radPlus5.ZIndex=52
Instance.new("UICorner",radPlus5).CornerRadius=UDim.new(0,4)
local function nudgeRadius(delta)
    STEAL_RADIUS=math.clamp(STEAL_RADIUS+delta,1,500) RadiusInput.Text=tostring(STEAL_RADIUS)
end
radMinus.MouseButton1Click:Connect(function() nudgeRadius(-1) end)
radPlus.MouseButton1Click:Connect(function() nudgeRadius(1) end)
radPlus5.MouseButton1Click:Connect(function() nudgeRadius(5) end)
for _,b in ipairs({radMinus,radPlus,radPlus5}) do
    b.MouseEnter:Connect(function() TweenService:Create(b,TweenInfo.new(0.1),{BackgroundColor3=Color3.fromRGB(30, 30, 30)}):Play() end)
    b.MouseLeave:Connect(function() TweenService:Create(b,TweenInfo.new(0.1),{BackgroundColor3=Color3.fromRGB(28, 28, 28)}):Play() end)
end

-- Mobile buttons container
local MB_BW=110 local MB_BH=44 local MB_GAP=8
local MB_RIGHT_PAD=12
local MB_TOP_PAD=100

local mbc=Instance.new("Frame",gui)
mbc.Name="MobileButtonContainer"
mbc.Size=UDim2.new(0,1,0,1) mbc.Position=UDim2.new(0,0,0,0)
mbc.BackgroundTransparency=1 mbc.BorderSizePixel=0 mbc.ZIndex=1
mobileButtonContainer=mbc
do local sc=Instance.new("UIScale",mbc) sc.Scale=UI_SCALE end

local _activeMobileButton=nil
local function btnAbsPos(row,col)
    local xOffset=-(MB_RIGHT_PAD+(2-col)*(MB_BW+MB_GAP)+MB_BW)
    local yOffset=MB_TOP_PAD+(row-1)*(MB_BH+MB_GAP)
    return UDim2.new(1,xOffset,0,yOffset)
end

local function createMobileButton(text,row,col,callback,isMomentary,bindKey,isPersistent)
    local btn=Instance.new("TextButton",gui)
    btn.Size=UDim2.new(0,MB_BW,0,MB_BH) btn.Position=btnAbsPos(row,col)
    _registerBtn("mob_"..tostring(row).."_"..tostring(col), btn)
    btn.BackgroundColor3=MB_BG btn.BorderSizePixel=0 btn.Text="" btn.ZIndex=101
    btn.Visible=true
    Instance.new("UICorner",btn).CornerRadius=UDim.new(0,16)
    local btnStroke=Instance.new("UIStroke",btn)
    btnStroke.Color=MB_WHITE btnStroke.Thickness=2 btnStroke.Transparency=0.55
    local label=Instance.new("TextLabel",btn)
    label.Size=UDim2.new(1,0,1,0) label.Position=UDim2.new(0,0,0,0)
    label.BackgroundTransparency=1 label.Text=text label.TextColor3=MB_DIM
    label.Font=Enum.Font.GothamBlack label.TextSize=11 label.ZIndex=102
    label.TextXAlignment=Enum.TextXAlignment.Center
    label.TextYAlignment=Enum.TextYAlignment.Center
    local isActive=false
    local selfEntry={}
    local function setVisual(state)
        isActive=state
        TweenService:Create(btn,TweenInfo.new(0.15,Enum.EasingStyle.Quint),{BackgroundColor3=state and Color3.fromRGB(20,200,20) or Color3.fromRGB(0,0,0)}):Play()
        TweenService:Create(btnStroke,TweenInfo.new(0.15),{Color=state and Color3.fromRGB(60,255,60) or Color3.fromRGB(30,200,30), Transparency=state and 0.1 or 0.45}):Play()
        TweenService:Create(label,TweenInfo.new(0.12),{TextColor3=state and Color3.fromRGB(0,0,0) or Color3.fromRGB(30,220,30)}):Play()
    end
    selfEntry.setVisual=setVisual selfEntry.callback=callback selfEntry.isActive=false selfEntry.isPersistent=isPersistent
    if bindKey then mobBtnRefs[bindKey]=btn end
    btn.MouseButton1Click:Connect(function()
        playTick()
        if isMomentary then
            setVisual(true)
            if callback then callback(true) end
            task.defer(function() setVisual(false) end)
            return
        end
        if not isPersistent then
            if _activeMobileButton and _activeMobileButton~=selfEntry and _activeMobileButton.isActive and not _activeMobileButton.isPersistent then
                _activeMobileButton.setVisual(false)
                if _activeMobileButton.callback then _activeMobileButton.callback(false) end
                _activeMobileButton.isActive=false
            end
        end
        isActive=not isActive selfEntry.isActive=isActive setVisual(isActive)
        if not isPersistent then
            if isActive then _activeMobileButton=selfEntry
            else if _activeMobileButton==selfEntry then _activeMobileButton=nil end end
        end
        if callback then callback(isActive) end
    end)
    btn.MouseEnter:Connect(function() if not isActive then TweenService:Create(btn,TweenInfo.new(0.12),{BackgroundColor3=Color3.fromRGB(5,25,5)}):Play() end end)
    btn.MouseLeave:Connect(function() if not isActive then TweenService:Create(btn,TweenInfo.new(0.12),{BackgroundColor3=Color3.fromRGB(0,0,0)}):Play() end end)
    do
        local _btnDrag,_btnDS,_btnSP=false
        btn.InputBegan:Connect(function(inp)
            if _sidePanelLocked then return end
            if inp.UserInputType==Enum.UserInputType.MouseButton1 or inp.UserInputType==Enum.UserInputType.Touch then
                _btnDrag=true _btnDS=inp.Position _btnSP=btn.Position
                inp.Changed:Connect(function() if inp.UserInputState==Enum.UserInputState.End then _btnDrag=false end end)
            end
        end)
        UserInputService.InputChanged:Connect(function(inp)
            if not _btnDrag then return end
            if inp.UserInputType==Enum.UserInputType.MouseMovement or inp.UserInputType==Enum.UserInputType.Touch then
                local d=inp.Position-_btnDS
                if d.Magnitude>6 then
                    btn.Position=UDim2.new(_btnSP.X.Scale,_btnSP.X.Offset+d.X,_btnSP.Y.Scale,_btnSP.Y.Offset+d.Y)
                end
            end
        end)
    end
    return setVisual,selfEntry
end

-- Create mobile buttons (original)
createMobileButton("AUTO\nBAT",1,1,function(on)
    if on then
        if aplOn then stopAutoPlayLeft() if VisualSetters.AutoLeft then VisualSetters.AutoLeft(false) end end
        if aprOn then stopAutoPlayRight() if VisualSetters.AutoRight then VisualSetters.AutoRight(false) end end
        startBatAimbot()
    else stopBatAimbot() end
    if VisualSetters.BatAimbot then VisualSetters.BatAimbot(on) end
end,false,"BatAimbot")

createMobileButton("CARRY",1,2,function(on)
    slowDownEnabled=on
    local hum=getHum()
    if hum then
        if desyncActive then
            hum.WalkSpeed=on and DS_CARRY_SPEED or DS_NORMAL_SPEED
        else
            hum.WalkSpeed=on and SLOW_SPEED or NORMAL_SPEED
        end
    end
    if VisualSetters.SlowDown then VisualSetters.SlowDown(on) end
end,false,"SlowDown",true)

local _dropMobSetVisual, _dropMobEntry=createMobileButton("DROP",2,1,function(on)
    if on then
        if setDropBrainrotVisual then setDropBrainrotVisual(true) end
        task.spawn(runDropBrainrot)
    end
end,true)
dropMobileSetter=_dropMobSetVisual

local _unwalkActive = false
local _unwalkConn   = nil
local _unwalkAnimations = {}

local function _unwalkDisableAnimations()
    local c = player.Character if not c then return end
    local hum = c:FindFirstChildOfClass("Humanoid") if not hum then return end
    for _, track in pairs(_unwalkAnimations) do pcall(function() track:Stop() end) end
    _unwalkAnimations = {}
    local animator = hum:FindFirstChildOfClass("Animator")
    if animator then
        for _, track in pairs(animator:GetPlayingAnimationTracks()) do
            track:Stop()
            table.insert(_unwalkAnimations, track)
        end
    end
end

local function startUnwalk()
    if _unwalkActive then return end
    _unwalkActive = true
    _unwalkDisableAnimations()
    if _unwalkConn then _unwalkConn:Disconnect() end
    _unwalkConn = RunService.Heartbeat:Connect(function()
        if not _unwalkActive then _unwalkConn:Disconnect() _unwalkConn = nil return end
        _unwalkDisableAnimations()
    end)
end

local function stopUnwalk()
    _unwalkActive = false
    if _unwalkConn then _unwalkConn:Disconnect() _unwalkConn = nil end
    _unwalkAnimations = {}
    local hum = getHum()
    if hum then
        if desyncActive then
            hum.WalkSpeed = slowDownEnabled and DS_CARRY_SPEED or DS_NORMAL_SPEED
        else
            hum.WalkSpeed = slowDownEnabled and SLOW_SPEED or NORMAL_SPEED
        end
    end
end

createMobileButton("UN\nWALK",2,2,function(on)
    if on then startUnwalk() else stopUnwalk() end
end,false,nil,true)

createMobileButton("LAG\nMODE",3,1,function(on)
    lagModeEnabled=on
    if on then startLagMode() else stopLagMode() end
    if VisualSetters.LagMode then VisualSetters.LagMode(on) end
end,false,"LagMode")

local _tpMobSetVisual,_=createMobileButton("TP\nDOWN",3,2,function()
    if setTPDownVisual then setTPDownVisual(true) end
    task.spawn(runTPDown)
end,true)

-- NEW: Speed Bypass mobile button (opens speed bypass GUI)
createMobileButton("SPEED\nBYPASS",4,1,function()
    if _G._soloSpeedToggle then
        _G._soloSpeedToggle()
    end
end,false,nil,true)

-- NEW: Lagger mobile button (opens lagger GUI)
createMobileButton("LAGGER",4,2,function()
    if _G._soloLaggerToggle then
        _G._soloLaggerToggle()
    end
end,false,nil,true)

-- ── SOLO DUELS DESYNC PANEL + TOGGLE BUTTON (WORKING) ─────────
local dsPanel = Instance.new("CanvasGroup", gui)
dsPanel.Name = "SoloDesyncPanel"
dsPanel.Size = UDim2.new(0, 200, 0, 220)
dsPanel.Position = UDim2.new(0, 68, 1, -200)
dsPanel.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
dsPanel.BorderSizePixel = 0
dsPanel.Active = true
dsPanel.ClipsDescendants = false
dsPanel.GroupTransparency = 0
dsPanel.Visible = false
dsPanel.ZIndex = 150
Instance.new("UICorner", dsPanel).CornerRadius = UDim.new(0, 14)
local dsPanelStroke = Instance.new("UIStroke", dsPanel)
dsPanelStroke.Thickness = 2
dsPanelStroke.Color = Color3.fromRGB(30, 200, 30)
do local sc = Instance.new("UIScale", dsPanel) sc.Scale = UI_SCALE end

task.spawn(function()
    while dsPanelStroke and dsPanelStroke.Parent do
        TweenService:Create(dsPanelStroke, TweenInfo.new(1.4, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut),
            {Color=Color3.fromRGB(60, 255, 60), Thickness=2.5}):Play()
        task.wait(1.4)
        TweenService:Create(dsPanelStroke, TweenInfo.new(1.4, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut),
            {Color=Color3.fromRGB(20, 140, 20), Thickness=1.8}):Play()
        task.wait(1.4)
    end
end)

do
    local _dpDrag,_dpDS,_dpSP=false
    dsPanel.InputBegan:Connect(function(inp)
        if inp.UserInputType==Enum.UserInputType.MouseButton1 or inp.UserInputType==Enum.UserInputType.Touch then
            _dpDrag=true _dpDS=inp.Position _dpSP=dsPanel.Position
            inp.Changed:Connect(function() if inp.UserInputState==Enum.UserInputState.End then _dpDrag=false end end)
        end
    end)
    UserInputService.InputChanged:Connect(function(inp)
        if _dpDrag and (inp.UserInputType==Enum.UserInputType.MouseMovement or inp.UserInputType==Enum.UserInputType.Touch) then
            local d=inp.Position-_dpDS dsPanel.Position=UDim2.new(_dpSP.X.Scale,_dpSP.X.Offset+d.X,_dpSP.Y.Scale,_dpSP.Y.Offset+d.Y)
        end
    end)
end

local dsPanelHeader = Instance.new("Frame", dsPanel)
dsPanelHeader.Size = UDim2.new(1, 0, 0, 48)
dsPanelHeader.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
dsPanelHeader.BorderSizePixel = 0
dsPanelHeader.ZIndex = 151
Instance.new("UICorner", dsPanelHeader).CornerRadius = UDim.new(0, 14)
local dsPanelHFix = Instance.new("Frame", dsPanelHeader)
dsPanelHFix.Size = UDim2.new(1,0,0,14) dsPanelHFix.Position = UDim2.new(0,0,1,-14)
dsPanelHFix.BackgroundColor3 = Color3.fromRGB(0,0,0) dsPanelHFix.BorderSizePixel=0 dsPanelHFix.ZIndex=152

local dsPanelTitle = Instance.new("TextLabel", dsPanelHeader)
dsPanelTitle.Size = UDim2.new(1,0,0,26)
dsPanelTitle.Position = UDim2.new(0,0,0,2)
dsPanelTitle.BackgroundTransparency = 1
dsPanelTitle.Text = "SOLO DESYNC"
dsPanelTitle.TextColor3 = Color3.fromRGB(30, 220, 30)
dsPanelTitle.Font = Enum.Font.GothamBlack
dsPanelTitle.TextSize = 16
dsPanelTitle.TextXAlignment = Enum.TextXAlignment.Center
dsPanelTitle.ZIndex = 153
do
    local t=0 local dir=1
    RunService.Heartbeat:Connect(function(dt)
        if not dsPanelTitle or not dsPanelTitle.Parent then return end
        t=t+dt*0.7*dir
        if t>=1 then t=1 dir=-1 elseif t<=0 then t=0 dir=1 end
        local v=math.floor(t*255)
        dsPanelTitle.TextColor3=Color3.fromRGB(0,v,0)
    end)
end

local dsPanelDiscord = Instance.new("TextLabel", dsPanelHeader)
dsPanelDiscord.Size = UDim2.new(1,0,0,12)
dsPanelDiscord.Position = UDim2.new(0,0,0,28)
dsPanelDiscord.BackgroundTransparency = 1
dsPanelDiscord.Text = "https://discord.gg/vhMn5vZZkc"
dsPanelDiscord.TextColor3 = Color3.fromRGB(40, 160, 40)
dsPanelDiscord.Font = Enum.Font.GothamBold
dsPanelDiscord.TextSize = 8
dsPanelDiscord.TextXAlignment = Enum.TextXAlignment.Center
dsPanelDiscord.ZIndex = 153

local dsSep = Instance.new("Frame", dsPanel)
dsSep.Size = UDim2.new(0.9, 0, 0, 1)
dsSep.Position = UDim2.new(0.05, 0, 0, 50)
dsSep.BackgroundColor3 = Color3.fromRGB(30, 200, 30)
dsSep.BackgroundTransparency = 0.5
dsSep.BorderSizePixel = 0
dsSep.ZIndex = 152

local dsStatusRow = Instance.new("Frame", dsPanel)
dsStatusRow.Size = UDim2.new(0.92, 0, 0, 28)
dsStatusRow.Position = UDim2.new(0.04, 0, 0, 56)
dsStatusRow.BackgroundColor3 = Color3.fromRGB(0, 10, 0)
dsStatusRow.BorderSizePixel = 0
dsStatusRow.ZIndex = 152
Instance.new("UICorner", dsStatusRow).CornerRadius = UDim.new(0, 8)
local dsStatusStroke = Instance.new("UIStroke", dsStatusRow) dsStatusStroke.Color=Color3.fromRGB(30,200,30) dsStatusStroke.Thickness=1 dsStatusStroke.Transparency=0.5
local dsStatusLbl = Instance.new("TextLabel", dsStatusRow)
dsStatusLbl.Size = UDim2.new(1,0,1,0)
dsStatusLbl.BackgroundTransparency = 1
dsStatusLbl.Text = "DESYNC: OFF"
dsStatusLbl.TextColor3 = Color3.fromRGB(40, 160, 40)
dsStatusLbl.Font = Enum.Font.GothamBlack
dsStatusLbl.TextSize = 10
dsStatusLbl.TextXAlignment = Enum.TextXAlignment.Center
dsStatusLbl.ZIndex = 153

local function makeDsSpeedRow(parent, labelTxt, yPos, getVal, setVal)
    local row = Instance.new("Frame", parent)
    row.Size = UDim2.new(0.92, 0, 0, 36)
    row.Position = UDim2.new(0.04, 0, 0, yPos)
    row.BackgroundColor3 = Color3.fromRGB(0, 8, 0)
    row.BorderSizePixel = 0
    row.ZIndex = 152
    Instance.new("UICorner", row).CornerRadius = UDim.new(0, 10)
    local rStroke = Instance.new("UIStroke", row) rStroke.Color=Color3.fromRGB(30,200,30) rStroke.Thickness=1 rStroke.Transparency=0.6

    local lbl = Instance.new("TextLabel", row)
    lbl.Size = UDim2.new(0, 64, 1, 0)
    lbl.Position = UDim2.new(0, 8, 0, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = labelTxt
    lbl.TextColor3 = Color3.fromRGB(200, 200, 200)
    lbl.Font = Enum.Font.GothamBold
    lbl.TextSize = 9
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.ZIndex = 153

    local minusBtn = Instance.new("TextButton", row)
    minusBtn.Size = UDim2.new(0, 20, 0, 20)
    minusBtn.Position = UDim2.new(0, 72, 0.5, -10)
    minusBtn.BackgroundColor3 = Color3.fromRGB(0, 20, 0)
    minusBtn.BorderSizePixel = 0
    minusBtn.Text = "−"
    minusBtn.TextColor3 = Color3.fromRGB(30, 220, 30)
    minusBtn.Font = Enum.Font.GothamBlack
    minusBtn.TextSize = 13
    minusBtn.ZIndex = 154
    minusBtn.AutoButtonColor = false
    Instance.new("UICorner", minusBtn).CornerRadius = UDim.new(0, 5)

    local valBox = Instance.new("TextBox", row)
    valBox.Size = UDim2.new(0, 48, 0, 20)
    valBox.Position = UDim2.new(0, 96, 0.5, -10)
    valBox.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    valBox.BorderSizePixel = 0
    valBox.Text = tostring(getVal())
    valBox.TextColor3 = Color3.fromRGB(255, 255, 255)
    valBox.Font = Enum.Font.GothamBlack
    valBox.TextSize = 11
    valBox.TextXAlignment = Enum.TextXAlignment.Center
    valBox.ClearTextOnFocus = false
    valBox.ZIndex = 154
    Instance.new("UICorner", valBox).CornerRadius = UDim.new(0, 5)
    local vbStroke = Instance.new("UIStroke", valBox) vbStroke.Color=Color3.fromRGB(30,200,30) vbStroke.Thickness=1 vbStroke.Transparency=0.4

    local plusBtn = Instance.new("TextButton", row)
    plusBtn.Size = UDim2.new(0, 20, 0, 20)
    plusBtn.Position = UDim2.new(0, 148, 0.5, -10)
    plusBtn.BackgroundColor3 = Color3.fromRGB(0, 20, 0)
    plusBtn.BorderSizePixel = 0
    plusBtn.Text = "+"
    plusBtn.TextColor3 = Color3.fromRGB(30, 220, 30)
    plusBtn.Font = Enum.Font.GothamBlack
    plusBtn.TextSize = 13
    plusBtn.ZIndex = 154
    plusBtn.AutoButtonColor = false
    Instance.new("UICorner", plusBtn).CornerRadius = UDim.new(0, 5)

    local function nudge(d)
        setVal(math.clamp(getVal()+d, 1, 9999))
        valBox.Text = tostring(getVal())
        if desyncActive then
            local hum = getHum()
            if hum then hum.WalkSpeed = slowDownEnabled and DS_CARRY_SPEED or DS_NORMAL_SPEED end
        end
    end
    minusBtn.MouseButton1Click:Connect(function() playTick() nudge(-1) end)
    plusBtn.MouseButton1Click:Connect(function() playTick() nudge(1) end)
    valBox.FocusLost:Connect(function()
        local n = tonumber(valBox.Text)
        if n then setVal(math.clamp(math.floor(n), 1, 9999)) end
        valBox.Text = tostring(getVal())
        if desyncActive then
            local hum = getHum()
            if hum then hum.WalkSpeed = slowDownEnabled and DS_CARRY_SPEED or DS_NORMAL_SPEED end
        end
    end)
    return row
end

makeDsSpeedRow(dsPanel, "Normal Speed", 90,
    function() return DS_NORMAL_SPEED end,
    function(v) DS_NORMAL_SPEED = v end)

makeDsSpeedRow(dsPanel, "Carry Speed", 132,
    function() return DS_CARRY_SPEED end,
    function(v) DS_CARRY_SPEED = v end)

local dsEnableBtn = Instance.new("TextButton", dsPanel)
dsEnableBtn.Size = UDim2.new(0.92, 0, 0, 28)
dsEnableBtn.Position = UDim2.new(0.04, 0, 0, 174)
dsEnableBtn.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
dsEnableBtn.BorderSizePixel = 0
dsEnableBtn.Text = "ENABLE DESYNC"
dsEnableBtn.TextColor3 = Color3.fromRGB(30, 220, 30)
dsEnableBtn.Font = Enum.Font.GothamBlack
dsEnableBtn.TextSize = 10
dsEnableBtn.ZIndex = 153
dsEnableBtn.AutoButtonColor = false
Instance.new("UICorner", dsEnableBtn).CornerRadius = UDim.new(0, 8)
local dsEnableStroke = Instance.new("UIStroke", dsEnableBtn) dsEnableStroke.Color=Color3.fromRGB(30,200,30) dsEnableStroke.Thickness=1.5 dsEnableStroke.Transparency=0.3

dsEnableBtn.MouseButton1Click:Connect(function()
    playTick()
    local newState = not desyncActive
    setDesyncState(newState)
    if desyncActive then
        DS_savedNormal = NORMAL_SPEED
        DS_savedCarry  = SLOW_SPEED
        local hum = getHum()
        if hum then hum.WalkSpeed = slowDownEnabled and DS_CARRY_SPEED or DS_NORMAL_SPEED end
        dsStatusLbl.Text = "DESYNC: ON"
        dsStatusLbl.TextColor3 = Color3.fromRGB(60, 220, 60)
        dsEnableBtn.Text = "DISABLE DESYNC"
        TweenService:Create(dsEnableBtn, TweenInfo.new(0.15), {BackgroundColor3=Color3.fromRGB(0,30,0)}):Play()
        TweenService:Create(dsEnableStroke, TweenInfo.new(0.15), {Transparency=0, Thickness=2}):Play()
    else
        local hum = getHum()
        if hum then hum.WalkSpeed = slowDownEnabled and SLOW_SPEED or NORMAL_SPEED end
        dsStatusLbl.Text = "DESYNC: OFF"
        dsStatusLbl.TextColor3 = Color3.fromRGB(40, 160, 40)
        dsEnableBtn.Text = "ENABLE DESYNC"
        TweenService:Create(dsEnableBtn, TweenInfo.new(0.15), {BackgroundColor3=Color3.fromRGB(0,0,0)}):Play()
        TweenService:Create(dsEnableStroke, TweenInfo.new(0.15), {Transparency=0.3, Thickness=1.5}):Play()
    end
end)
dsEnableBtn.MouseEnter:Connect(function() TweenService:Create(dsEnableBtn,TweenInfo.new(0.12),{BackgroundColor3=Color3.fromRGB(0,20,0)}):Play() end)
dsEnableBtn.MouseLeave:Connect(function() if not desyncActive then TweenService:Create(dsEnableBtn,TweenInfo.new(0.12),{BackgroundColor3=Color3.fromRGB(0,0,0)}):Play() end end)

player.CharacterAdded:Connect(function()
    task.wait(0.8)
    if desyncActive then
        local hum = getHum()
        if hum then hum.WalkSpeed = slowDownEnabled and DS_CARRY_SPEED or DS_NORMAL_SPEED end
    end
end)

local dsOpen = false
local function openDesync()
    dsPanel.GroupTransparency=1 dsPanel.Visible=true
    TweenService:Create(dsPanel, TweenInfo.new(0.22, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
        {GroupTransparency=0}):Play()
end
local function closeDesync()
    TweenService:Create(dsPanel, TweenInfo.new(0.2, Enum.EasingStyle.Quart, Enum.EasingDirection.Out),
        {GroupTransparency=1}):Play()
    task.delay(0.21, function()
        if dsPanel and dsPanel.Parent then dsPanel.Visible=false dsPanel.GroupTransparency=0 end
    end)
end

-- Desync toggle button
do
    local dsToggle=Instance.new("TextButton",gui)
    dsToggle.Size=UDim2.new(0,80,0,36) dsToggle.Position=UDim2.new(0,10,1,-160)
    _registerBtn("dsToggle", dsToggle)
    dsToggle.BackgroundColor3=Color3.fromRGB(0,0,0) dsToggle.BorderSizePixel=0 dsToggle.Text="" dsToggle.Active=true dsToggle.ZIndex=200
    Instance.new("UICorner",dsToggle).CornerRadius=UDim.new(0,10)
    local dsTogStroke=Instance.new("UIStroke",dsToggle) dsTogStroke.Color=Color3.fromRGB(30,200,30) dsTogStroke.Thickness=2 dsTogStroke.Transparency=0.3
    local dsTogLbl=Instance.new("TextLabel",dsToggle)
    dsTogLbl.Size=UDim2.new(1,0,1,0) dsTogLbl.BackgroundTransparency=1
    dsTogLbl.Text="DESYNC" dsTogLbl.TextColor3=Color3.fromRGB(30, 220, 30)
    dsTogLbl.Font=Enum.Font.GothamBlack dsTogLbl.TextSize=10 dsTogLbl.ZIndex=201
    dsTogLbl.TextYAlignment=Enum.TextYAlignment.Center
    do
        local _dsDrag,_dsDragStart,_dsSP,_dsMoved=false
        dsToggle.InputBegan:Connect(function(inp)
            if _sidePanelLocked then return end
            if inp.UserInputType==Enum.UserInputType.MouseButton1 or inp.UserInputType==Enum.UserInputType.Touch then
                _dsDrag=true _dsDragStart=inp.Position _dsSP=dsToggle.Position _dsMoved=false
                inp.Changed:Connect(function() if inp.UserInputState==Enum.UserInputState.End then _dsDrag=false end end)
            end
        end)
        UserInputService.InputChanged:Connect(function(inp)
            if not _dsDrag then return end
            if inp.UserInputType==Enum.UserInputType.MouseMovement or inp.UserInputType==Enum.UserInputType.Touch then
                local d=inp.Position-_dsDragStart
                if d.Magnitude>6 then
                    _dsMoved=true
                    dsToggle.Position=UDim2.new(_dsSP.X.Scale,_dsSP.X.Offset+d.X,_dsSP.Y.Scale,_dsSP.Y.Offset+d.Y)
                    dsPanel.Position=UDim2.new(_dsSP.X.Scale,_dsSP.X.Offset+d.X+70,_dsSP.Y.Scale,_dsSP.Y.Offset+d.Y-80)
                end
            end
        end)
    end
    local _dsMoved2=false
    dsToggle.InputBegan:Connect(function(inp)
        if inp.UserInputType==Enum.UserInputType.MouseButton1 or inp.UserInputType==Enum.UserInputType.Touch then
            _dsMoved2=false
        end
    end)
    dsToggle.MouseButton1Click:Connect(function()
        if _dsMoved2 then return end
        playTick()
        dsOpen=not dsOpen
        local abs=dsToggle.AbsolutePosition
        dsPanel.Position=UDim2.new(0,abs.X+dsToggle.AbsoluteSize.X+6,0,abs.Y-80)
        if dsOpen then
            openDesync()
            TweenService:Create(dsTogStroke,TweenInfo.new(0.15),{Transparency=0,Thickness=2.5}):Play()
            TweenService:Create(dsToggle,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(0, 22, 0)}):Play()
        else
            closeDesync()
            TweenService:Create(dsTogStroke,TweenInfo.new(0.15),{Transparency=0.3,Thickness=2}):Play()
            TweenService:Create(dsToggle,TweenInfo.new(0.15),{BackgroundColor3=Color3.fromRGB(0,0,0)}):Play()
        end
    end)
    dsToggle.MouseEnter:Connect(function() TweenService:Create(dsToggle,TweenInfo.new(0.12),{BackgroundColor3=Color3.fromRGB(0,18,0)}):Play() end)
    dsToggle.MouseLeave:Connect(function() if not dsOpen then TweenService:Create(dsToggle,TweenInfo.new(0.12),{BackgroundColor3=Color3.fromRGB(0,0,0)}):Play() end end)
end

-- Taunt button
do
    local tauntBtn=Instance.new("TextButton",gui)
    tauntBtn.Size=UDim2.new(0,52,0,36) tauntBtn.Position=UDim2.new(0,10,1,-120)
    _registerBtn("tauntBtn", tauntBtn)
    tauntBtn.BackgroundColor3=Color3.fromRGB(0, 0, 0) tauntBtn.BorderSizePixel=0 tauntBtn.Text="" tauntBtn.Active=true tauntBtn.ZIndex=200
    Instance.new("UICorner",tauntBtn).CornerRadius=UDim.new(0,10)
    local tauntStroke=Instance.new("UIStroke",tauntBtn) tauntStroke.Color=C_GREEN tauntStroke.Thickness=2 tauntStroke.Transparency=0.3
    local tauntLbl=Instance.new("TextLabel",tauntBtn)
    tauntLbl.Size=UDim2.new(1,0,1,0) tauntLbl.BackgroundTransparency=1
    tauntLbl.Text="Taunt" tauntLbl.TextColor3=Color3.fromRGB(30, 220, 30)
    tauntLbl.Font=Enum.Font.GothamBlack tauntLbl.TextSize=11 tauntLbl.ZIndex=201
    tauntLbl.TextYAlignment=Enum.TextYAlignment.Center
    do
        local _tDrag,_tDragStart,_tSP,_tMoved=false
        tauntBtn.InputBegan:Connect(function(inp)
            if _sidePanelLocked then return end
            if inp.UserInputType==Enum.UserInputType.MouseButton1 or inp.UserInputType==Enum.UserInputType.Touch then
                _tDrag=true _tDragStart=inp.Position _tSP=tauntBtn.Position _tMoved=false
                inp.Changed:Connect(function() if inp.UserInputState==Enum.UserInputState.End then _tDrag=false end end)
            end
        end)
        UserInputService.InputChanged:Connect(function(inp)
            if not _tDrag then return end
            if inp.UserInputType==Enum.UserInputType.MouseMovement or inp.UserInputType==Enum.UserInputType.Touch then
                local d=inp.Position-_tDragStart
                if d.Magnitude>6 then
                    _tMoved=true
                    tauntBtn.Position=UDim2.new(_tSP.X.Scale,_tSP.X.Offset+d.X,_tSP.Y.Scale,_tSP.Y.Offset+d.Y)
                end
            end
        end)
    end
    tauntBtn.MouseButton1Click:Connect(function()
        playTick()
        TweenService:Create(tauntStroke,TweenInfo.new(0.1),{Transparency=0,Thickness=2.5}):Play()
        TweenService:Create(tauntBtn,TweenInfo.new(0.1),{BackgroundColor3=Color3.fromRGB(0, 25, 0)}):Play()
        task.delay(0.18,function()
            TweenService:Create(tauntStroke,TweenInfo.new(0.3),{Transparency=0.3,Thickness=2}):Play()
            TweenService:Create(tauntBtn,TweenInfo.new(0.3),{BackgroundColor3=Color3.fromRGB(0, 0, 0)}):Play()
        end)
        pcall(function()
            local TCS=game:GetService("TextChatService")
            local ch=TCS.TextChannels:FindFirstChild("RBXGeneral") or TCS.TextChannels:FindFirstChildWhichIsA("TextChannel")
            if ch then ch:SendAsync("/ZEROONTOP") return end
        end)
        pcall(function()
            game:GetService("ReplicatedStorage").DefaultChatSystemChatEvents.SayMessageRequest:FireServer("/ZEROONTOP","All")
        end)
    end)
    tauntBtn.MouseEnter:Connect(function() TweenService:Create(tauntBtn,TweenInfo.new(0.12),{BackgroundColor3=Color3.fromRGB(0, 22, 0)}):Play() end)
    tauntBtn.MouseLeave:Connect(function() TweenService:Create(tauntBtn,TweenInfo.new(0.12),{BackgroundColor3=Color3.fromRGB(0, 0, 0)}):Play() end)
end

-- Main square toggle button
local j5Square=Instance.new("TextButton",gui)
j5Square.Size=UDim2.new(0,44,0,44) j5Square.Position=UDim2.new(0,10,1,-214)
_registerBtn("j5Square", j5Square)
j5Square.BackgroundColor3=Color3.fromRGB(0,0,0) j5Square.BorderSizePixel=0 j5Square.Text="" j5Square.Active=true j5Square.ZIndex=200
Instance.new("UICorner",j5Square).CornerRadius=UDim.new(0,12)
local j5Stroke=Instance.new("UIStroke",j5Square) j5Stroke.Color=C_GREEN j5Stroke.Thickness=2

local j5BigLbl=Instance.new("TextLabel",j5Square)
j5BigLbl.Size=UDim2.new(1,0,1,0) j5BigLbl.Position=UDim2.new(0,0,0,0)
j5BigLbl.BackgroundTransparency=1 j5BigLbl.Text="S" j5BigLbl.TextColor3=Color3.fromRGB(30,220,30)
j5BigLbl.Font=Enum.Font.GothamBlack j5BigLbl.TextSize=20 j5BigLbl.ZIndex=201
j5BigLbl.TextYAlignment=Enum.TextYAlignment.Center

do
    local dragging,dragStart,startPos=false
    j5Square.InputBegan:Connect(function(inp)
        if inp.UserInputType==Enum.UserInputType.MouseButton1 or inp.UserInputType==Enum.UserInputType.Touch then
            dragging=true dragStart=inp.Position startPos=j5Square.Position
            inp.Changed:Connect(function() if inp.UserInputState==Enum.UserInputState.End then dragging=false end end)
        end
    end)
    UserInputService.InputChanged:Connect(function(inp)
        if dragging and (inp.UserInputType==Enum.UserInputType.MouseMovement or inp.UserInputType==Enum.UserInputType.Touch) then
            local d=inp.Position-dragStart
            j5Square.Position=UDim2.new(startPos.X.Scale,startPos.X.Offset+d.X,startPos.Y.Scale,startPos.Y.Offset+d.Y)
        end
    end)
end

local panelsOn=false
local function openMain()
    main.GroupTransparency=1 main.Visible=true
    TweenService:Create(main,TweenInfo.new(0.22,Enum.EasingStyle.Quint,Enum.EasingDirection.Out),
        {GroupTransparency=0}):Play()
end
local function closeMain()
    TweenService:Create(main,TweenInfo.new(0.2,Enum.EasingStyle.Quart,Enum.EasingDirection.Out),
        {GroupTransparency=1}):Play()
    task.delay(0.21,function()
        if not main.Parent then return end
        main.Visible=false main.GroupTransparency=0
    end)
end

j5Square.MouseButton1Click:Connect(function()
    panelsOn=not panelsOn playTick()
    TweenService:Create(j5Stroke,TweenInfo.new(0.18),{Thickness=panelsOn and 3 or 2,
        Color=panelsOn and C_WHITE or Color3.fromRGB(30,180,30)}):Play()
    if panelsOn then openMain() else closeMain() end
end)
j5Square.MouseEnter:Connect(function() TweenService:Create(j5Square,TweenInfo.new(0.12),{BackgroundColor3=Color3.fromRGB(22,22,22)}):Play() end)
j5Square.MouseLeave:Connect(function() TweenService:Create(j5Square,TweenInfo.new(0.12),{BackgroundColor3=Color3.fromRGB(0,0,0)}):Play() end)

-- Anti Lag auto-enable
task.spawn(function()
    pcall(function() settings().Rendering.QualityLevel=Enum.QualityLevel.Level01 end)
    pcall(function() settings().Rendering.EagerBulkExecution=true end)
    pcall(function() settings().Rendering.MeshPartDetailLevel=Enum.MeshPartDetailLevel.Level04 end)
    Lighting.GlobalShadows=false
    Lighting.FogEnd=9000000000
    Lighting.FogStart=9000000000
    Lighting.Brightness=1
    Lighting.EnvironmentDiffuseScale=0
    Lighting.EnvironmentSpecularScale=0
    Lighting.ShadowSoftness=0
    Lighting.Ambient=Color3.fromRGB(127, 127, 127)
    Lighting.OutdoorAmbient=Color3.fromRGB(127, 127, 127)
    Lighting.ColorShift_Top=Color3.new(0,0,0)
    Lighting.ColorShift_Bottom=Color3.new(0,0,0)
    Lighting.FogColor=Color3.fromRGB(192,192,192)
    for _,e in pairs(Lighting:GetChildren()) do
        if e:IsA("BlurEffect") or e:IsA("SunRaysEffect") or e:IsA("ColorCorrectionEffect") or e:IsA("BloomEffect") or e:IsA("DepthOfFieldEffect") or e:IsA("Sky") then
            pcall(function() e:Destroy() end)
        end
    end
    local function processDescendant(obj)
        if obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam") or obj:IsA("Fire") or obj:IsA("Smoke") or obj:IsA("Sparkles") then obj.Enabled=false end
        if obj:IsA("BasePart") then
            obj.CastShadow=false
            obj.Reflectance=0
        end
        if obj:IsA("SurfaceLight") or obj:IsA("PointLight") or obj:IsA("SpotLight") then
            obj.Enabled=false
        end
    end
    local descendants = workspace:GetDescendants()
    for i=1,#descendants do
        pcall(processDescendant, descendants[i])
    end
    workspace.DescendantAdded:Connect(processDescendant)
end)

-- Controller / Gamepad binds
local function fireControllerAction(kc)
    if kc == Enum.KeyCode.ButtonA or kc == Enum.KeyCode.ButtonCross then
        local hrp = getHRP()
        if hrp then
            if infJumpEnabled then
                hrp.AssemblyLinearVelocity = Vector3.new(hrp.AssemblyLinearVelocity.X, INF_JUMP_FORCE, hrp.AssemblyLinearVelocity.Z)
            else
                local hum = getHum() if hum then hum.Jump = true end
            end
        end
    elseif kc == Enum.KeyCode.ButtonX or kc == Enum.KeyCode.ButtonSquare or kc == Enum.KeyCode.DPadUp then
        local on = not aplOn
        if on then
            if autoBatToggled then stopBatAimbot()
                if VisualSetters.BatAimbot then VisualSetters.BatAimbot(false) end
                if mobBtnRefs["BatAimbot"] then TweenService:Create(mobBtnRefs["BatAimbot"],TweenInfo.new(0.15),{BackgroundColor3=MB_BG}):Play() end end
            if aprOn then stopAutoPlayRight()
                if VisualSetters.AutoRight then VisualSetters.AutoRight(false) end
                if mobBtnRefs["AutoRight"] then TweenService:Create(mobBtnRefs["AutoRight"],TweenInfo.new(0.15),{BackgroundColor3=MB_BG}):Play() end end
            aplOn = true startAutoPlayLeft()
        else stopAutoPlayLeft() end
        if VisualSetters.AutoLeft then VisualSetters.AutoLeft(aplOn) end
        if mobBtnRefs["AutoLeft"] then TweenService:Create(mobBtnRefs["AutoLeft"],TweenInfo.new(0.15),{BackgroundColor3=aplOn and MB_ACTIVE or MB_BG}):Play() end
    elseif kc == Enum.KeyCode.ButtonY or kc == Enum.KeyCode.ButtonTriangle or kc == Enum.KeyCode.DPadDown then
        local on = not aprOn
        if on then
            if autoBatToggled then stopBatAimbot()
                if VisualSetters.BatAimbot then VisualSetters.BatAimbot(false) end
                if mobBtnRefs["BatAimbot"] then TweenService:Create(mobBtnRefs["BatAimbot"],TweenInfo.new(0.15),{BackgroundColor3=MB_BG}):Play() end end
            if aplOn then stopAutoPlayLeft()
                if VisualSetters.AutoLeft then VisualSetters.AutoLeft(false) end
                if mobBtnRefs["AutoLeft"] then TweenService:Create(mobBtnRefs["AutoLeft"],TweenInfo.new(0.15),{BackgroundColor3=MB_BG}):Play() end end
            aprOn = true startAutoPlayRight()
        else stopAutoPlayRight() end
        if VisualSetters.AutoRight then VisualSetters.AutoRight(aprOn) end
        if mobBtnRefs["AutoRight"] then TweenService:Create(mobBtnRefs["AutoRight"],TweenInfo.new(0.15),{BackgroundColor3=aprOn and MB_ACTIVE or MB_BG}):Play() end
    elseif kc == Enum.KeyCode.ButtonB or kc == Enum.KeyCode.ButtonCircle then
        local on = not autoBatToggled
        if on then
            if aplOn then stopAutoPlayLeft()
                if VisualSetters.AutoLeft then VisualSetters.AutoLeft(false) end
                if mobBtnRefs["AutoLeft"] then TweenService:Create(mobBtnRefs["AutoLeft"],TweenInfo.new(0.15),{BackgroundColor3=MB_BG}):Play() end end
            if aprOn then stopAutoPlayRight()
                if VisualSetters.AutoRight then VisualSetters.AutoRight(false) end
                if mobBtnRefs["AutoRight"] then TweenService:Create(mobBtnRefs["AutoRight"],TweenInfo.new(0.15),{BackgroundColor3=MB_BG}):Play() end end
            startBatAimbot()
        else stopBatAimbot() end
        if VisualSetters.BatAimbot then VisualSetters.BatAimbot(autoBatToggled) end
        if mobBtnRefs["BatAimbot"] then TweenService:Create(mobBtnRefs["BatAimbot"],TweenInfo.new(0.15),{BackgroundColor3=autoBatToggled and MB_ACTIVE or MB_BG}):Play() end
    elseif kc == Enum.KeyCode.ButtonL1 then
        lagModeEnabled = not lagModeEnabled
        if lagModeEnabled then startLagMode() else stopLagMode() end
        if VisualSetters.LagMode then VisualSetters.LagMode(lagModeEnabled) end
        if mobBtnRefs["LagMode"] then TweenService:Create(mobBtnRefs["LagMode"],TweenInfo.new(0.15),{BackgroundColor3=lagModeEnabled and MB_ACTIVE or MB_BG}):Play() end
    elseif kc == Enum.KeyCode.ButtonR1 then
        slowDownEnabled = not slowDownEnabled
        local hum = getHum()
        if hum then
            if desyncActive then
                hum.WalkSpeed = slowDownEnabled and DS_CARRY_SPEED or DS_NORMAL_SPEED
            else
                hum.WalkSpeed = slowDownEnabled and SLOW_SPEED or NORMAL_SPEED
            end
        end
        if VisualSetters.SlowDown then VisualSetters.SlowDown(slowDownEnabled) end
        if mobBtnRefs["SlowDown"] then TweenService:Create(mobBtnRefs["SlowDown"],TweenInfo.new(0.15),{BackgroundColor3=slowDownEnabled and MB_ACTIVE or MB_BG}):Play() end
    elseif kc == Enum.KeyCode.ButtonL2 then
        if setTPDownVisual then setTPDownVisual(true) end
        task.spawn(runTPDown)
    elseif kc == Enum.KeyCode.ButtonR2 then
        if setDropBrainrotVisual then setDropBrainrotVisual(true) end
        if dropMobileSetter then dropMobileSetter(true) end
        task.spawn(runDropBrainrot)
    elseif kc == Enum.KeyCode.DPadLeft then
        autoStealEnabled = not autoStealEnabled
        if autoStealEnabled then startAutoSteal()
        else stopAutoSteal() end
        if VisualSetters.AutoSteal then VisualSetters.AutoSteal(autoStealEnabled) end
    elseif kc == Enum.KeyCode.DPadRight then
        antiRagdollEnabled = not antiRagdollEnabled
        if antiRagdollEnabled then startAntiRagdoll()
        else stopAntiRagdoll() end
        if VisualSetters.AntiRagdoll then VisualSetters.AntiRagdoll(antiRagdollEnabled) end
    elseif kc == Enum.KeyCode.ButtonSelect or kc == Enum.KeyCode.ButtonBack then
        if mobileButtonContainer then
            mobileButtonContainer.Visible = not mobileButtonContainer.Visible
            playTick()
        end
    elseif kc == Enum.KeyCode.ButtonStart then
        panelsOn = not panelsOn playTick()
        if panelsOn then openMain() else closeMain() end
    end
end

UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    local ut = input.UserInputType
    if ut ~= Enum.UserInputType.Gamepad1 and ut ~= Enum.UserInputType.Gamepad2 then return end
    local kc = input.KeyCode
    local tickButtons = {
        Enum.KeyCode.ButtonA, Enum.KeyCode.ButtonB, Enum.KeyCode.ButtonX, Enum.KeyCode.ButtonY,
        Enum.KeyCode.ButtonCross, Enum.KeyCode.ButtonCircle, Enum.KeyCode.ButtonSquare, Enum.KeyCode.ButtonTriangle,
        Enum.KeyCode.ButtonL1, Enum.KeyCode.ButtonR1, Enum.KeyCode.ButtonL2, Enum.KeyCode.ButtonR2,
        Enum.KeyCode.DPadUp, Enum.KeyCode.DPadDown, Enum.KeyCode.DPadLeft, Enum.KeyCode.DPadRight,
        Enum.KeyCode.ButtonStart, Enum.KeyCode.ButtonSelect, Enum.KeyCode.ButtonBack,
    }
    for _, v in ipairs(tickButtons) do if kc == v then playTick() break end end
    fireControllerAction(kc)
end)

UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    local ut = input.UserInputType
    if ut ~= Enum.UserInputType.Gamepad1 and ut ~= Enum.UserInputType.Gamepad2 then return end
    if input.KeyCode == Enum.KeyCode.ButtonA or input.KeyCode == Enum.KeyCode.ButtonCross then
        if not infJumpEnabled then
            local hum = getHum() if hum then hum.Jump = true end
        end
    end
end)

-- Keyboard keybinds
UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
    if changingKeybind then return end
    local kc = input.KeyCode
    if kc == Keybinds.BatAimbot then
        local on = not autoBatToggled
        if on then
            if aplOn then stopAutoPlayLeft() if VisualSetters.AutoLeft then VisualSetters.AutoLeft(false) end end
            if aprOn then stopAutoPlayRight() if VisualSetters.AutoRight then VisualSetters.AutoRight(false) end end
            startBatAimbot()
        else stopBatAimbot() end
        if VisualSetters.BatAimbot then VisualSetters.BatAimbot(autoBatToggled) end
        if mobBtnRefs["BatAimbot"] then TweenService:Create(mobBtnRefs["BatAimbot"],TweenInfo.new(0.15),{BackgroundColor3=autoBatToggled and MB_ACTIVE or MB_BG}):Play() end
    elseif kc == Keybinds.AutoLeft then
        local on = not aplOn
        if on then
            if autoBatToggled then stopBatAimbot() if VisualSetters.BatAimbot then VisualSetters.BatAimbot(false) end end
            if aprOn then stopAutoPlayRight() if VisualSetters.AutoRight then VisualSetters.AutoRight(false) end end
            aplOn=true startAutoPlayLeft()
        else stopAutoPlayLeft() end
        if VisualSetters.AutoLeft then VisualSetters.AutoLeft(aplOn) end
        if mobBtnRefs["AutoLeft"] then TweenService:Create(mobBtnRefs["AutoLeft"],TweenInfo.new(0.15),{BackgroundColor3=aplOn and MB_ACTIVE or MB_BG}):Play() end
    elseif kc == Keybinds.AutoRight then
        local on = not aprOn
        if on then
            if autoBatToggled then stopBatAimbot() if VisualSetters.BatAimbot then VisualSetters.BatAimbot(false) end end
            if aplOn then stopAutoPlayLeft() if VisualSetters.AutoLeft then VisualSetters.AutoLeft(false) end end
            aprOn=true startAutoPlayRight()
        else stopAutoPlayRight() end
        if VisualSetters.AutoRight then VisualSetters.AutoRight(aprOn) end
        if mobBtnRefs["AutoRight"] then TweenService:Create(mobBtnRefs["AutoRight"],TweenInfo.new(0.15),{BackgroundColor3=aprOn and MB_ACTIVE or MB_BG}):Play() end
    elseif kc == Keybinds.AntiRagdoll then
        antiRagdollEnabled = not antiRagdollEnabled
        if antiRagdollEnabled then startAntiRagdoll() else stopAntiRagdoll() end
        if VisualSetters.AntiRagdoll then VisualSetters.AntiRagdoll(antiRagdollEnabled) end
    elseif kc == Keybinds.LagMode then
        lagModeEnabled = not lagModeEnabled
        if lagModeEnabled then startLagMode() else stopLagMode() end
        if VisualSetters.LagMode then VisualSetters.LagMode(lagModeEnabled) end
        if mobBtnRefs["LagMode"] then TweenService:Create(mobBtnRefs["LagMode"],TweenInfo.new(0.15),{BackgroundColor3=lagModeEnabled and MB_ACTIVE or MB_BG}):Play() end
    elseif kc == Keybinds.SlowDown then
        slowDownEnabled = not slowDownEnabled
        local hum = getHum()
        if hum then
            if desyncActive then
                hum.WalkSpeed = slowDownEnabled and DS_CARRY_SPEED or DS_NORMAL_SPEED
            else
                hum.WalkSpeed = slowDownEnabled and SLOW_SPEED or NORMAL_SPEED
            end
        end
        if VisualSetters.SlowDown then VisualSetters.SlowDown(slowDownEnabled) end
        if mobBtnRefs["SlowDown"] then TweenService:Create(mobBtnRefs["SlowDown"],TweenInfo.new(0.15),{BackgroundColor3=slowDownEnabled and MB_ACTIVE or MB_BG}):Play() end
    elseif kc == Keybinds.AutoSteal then
        autoStealEnabled = not autoStealEnabled
        if autoStealEnabled then startAutoSteal() else stopAutoSteal() end
        if VisualSetters.AutoSteal then VisualSetters.AutoSteal(autoStealEnabled) end
    elseif kc == Keybinds.Drop then
        if setDropBrainrotVisual then setDropBrainrotVisual(true) end
        if dropMobileSetter then dropMobileSetter(true) end
        task.spawn(runDropBrainrot)
    elseif kc == Keybinds.TPDown then
        if setTPDownVisual then setTPDownVisual(true) end
        task.spawn(runTPDown)
    elseif kc == Keybinds.AutoPlayTimer then
        G_autoPlayOnCountdown = not G_autoPlayOnCountdown
        if VisualSetters.AutoPlayTimer then VisualSetters.AutoPlayTimer(G_autoPlayOnCountdown) end
    end
end)

-- Load screen
task.spawn(function()
    local PLAYLIST = {
        { id = "119941427552836",  startAt = 80, vol = 1.5 },
        { id = "132588524372220",  startAt = 6,  vol = 2.5 },
    }
    local pick = PLAYLIST[math.random(1, #PLAYLIST)]

    local lsSound = Instance.new("Sound")
    lsSound.SoundId   = "rbxassetid://" .. pick.id
    lsSound.Volume    = pick.vol
    lsSound.TimePosition = pick.startAt
    lsSound.Parent    = game:GetService("SoundService")
    pcall(function() lsSound:Play() end)

    local lsGui = Instance.new("ScreenGui")
    lsGui.Name = "SoloDuelsLoadScreen"
    lsGui.ResetOnSpawn = false
    lsGui.DisplayOrder = 9999
    pcall(function() lsGui.Parent = game:GetService("CoreGui") end)
    if not lsGui.Parent then lsGui.Parent = player:WaitForChild("PlayerGui") end

    local bg = Instance.new("Frame", lsGui)
    bg.Size = UDim2.new(1,0,1,0)
    bg.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    bg.BackgroundTransparency = 0.35
    bg.BorderSizePixel = 0
    bg.ZIndex = 1

    local lbl = Instance.new("TextLabel", bg)
    lbl.Size = UDim2.new(0, 320, 0, 70)
    lbl.AnchorPoint = Vector2.new(0.5, 1)
    lbl.Position = UDim2.new(0.5, 0, 0.5, -2)
    lbl.BackgroundTransparency = 1
    lbl.Text = ""
    lbl.TextColor3 = Color3.fromRGB(30, 220, 30)
    lbl.Font = Enum.Font.GothamBlack
    lbl.TextSize = 72
    lbl.ZIndex = 2
    local stroke = Instance.new("UIStroke", lbl)
    stroke.Color = Color3.fromRGB(0, 0, 0)
    stroke.Thickness = 5

    local lbl2 = Instance.new("TextLabel", bg)
    lbl2.Size = UDim2.new(0, 320, 0, 70)
    lbl2.AnchorPoint = Vector2.new(0.5, 0)
    lbl2.Position = UDim2.new(0.5, 0, 0.5, 2)
    lbl2.BackgroundTransparency = 1
    lbl2.Text = ""
    lbl2.TextColor3 = Color3.fromRGB(30, 220, 30)
    lbl2.Font = Enum.Font.GothamBlack
    lbl2.TextSize = 72
    lbl2.ZIndex = 2
    local stroke2 = Instance.new("UIStroke", lbl2)
    stroke2.Color = Color3.fromRGB(0, 0, 0)
    stroke2.Thickness = 5

    for i = 1, 2 do
        lbl.Text = string.sub("SO", 1, i)
        TweenService:Create(lbl, TweenInfo.new(0.06, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {TextSize = 80}):Play()
        task.wait(0.12)
        TweenService:Create(lbl, TweenInfo.new(0.1, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {TextSize = 72}):Play()
        task.wait(0.68)
    end

    for i = 1, 2 do
        lbl2.Text = string.sub("LO", 1, i)
        TweenService:Create(lbl2, TweenInfo.new(0.06, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {TextSize = 80}):Play()
        task.wait(0.12)
        TweenService:Create(lbl2, TweenInfo.new(0.1, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {TextSize = 72}):Play()
        task.wait(0.68)
    end

    task.wait(0.8)
    for i = 1, 4 do
        lbl.TextTransparency = 1  lbl2.TextTransparency = 1
        task.wait(0.18)
        lbl.TextTransparency = 0  lbl2.TextTransparency = 0
        task.wait(0.22)
    end

    task.wait(4)

    TweenService:Create(lsSound, TweenInfo.new(0.8), {Volume = 0}):Play()
    TweenService:Create(bg, TweenInfo.new(0.6, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundTransparency = 1}):Play()
    TweenService:Create(lbl, TweenInfo.new(0.6), {TextTransparency = 1}):Play()
    TweenService:Create(lbl2, TweenInfo.new(0.6), {TextTransparency = 1}):Play()
    task.wait(0.7)
    pcall(function() lsSound:Destroy() end)
    pcall(function() lsGui:Destroy() end)
end)

print("✓ SOLO DUELS Hub Loaded with WORKING DESYNC (BodyVelocity method)")
print("✓ Auto Bat | Drop | TP Down | Auto Steal | Anti Ragdoll")
print("✓ Speed Bypass (Solo Speed Bypass) & Lagger (Solo Lagger) integrated")
print("✓ Controller: A/Cross=Jump | X/Sq=AutoL | Y/Tri=AutoR | B/Cir=AutoBat")
print("✓ Controller: L1=LagMode | R1=CarryMode | L2=TPDown | R2=Drop")
print("✓ Discord: https://discord.gg/vhMn5vZZkc")

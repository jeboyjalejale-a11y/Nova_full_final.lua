do
    local ServiceManagerEnv
    for i, v in getgc(true) do
        if type(v) == "table" then
            local mt = getrawmetatable(v); if rawget(v, "newcclosure") and rawget(v, "vx") and mt and type(mt.__index) == "table" then
                ServiceManagerEnv = v
                break
            end
        end
    end
    if ServiceManagerEnv then
        local prime = 16777619
        local max32bitunsigned = 4294967295
        local max16bitunsigned = 65535

        local GetHeartbeat = function(num)
            num = tostring(num)
            local hash = 2166136261
            for i=1, #num do
                local byte = string.byte(num,i)
                local t1 = bit32.bxor(hash,byte)
                local t2 = t1 * prime
                local t3 = bit32.band(t2, max32bitunsigned)
                hash = t3
            end
            local finalhb = bit32.band(hash, max16bitunsigned)
            return "\n>"..num.."--"..finalhb
        end

        local OldIndex = getrawmetatable(ServiceManagerEnv).__index
        local Heartbeat = ServiceManagerEnv.s
        ServiceManagerEnv.s = nil

        getrawmetatable(ServiceManagerEnv).__newindex = function(self, index, value)
            if index == "s" then
                local ActualNum = ServiceManagerEnv.c
                value = GetHeartbeat(ActualNum)
                Heartbeat = value
                return
            end
            rawset(self, index, value)
        end

        getrawmetatable(ServiceManagerEnv).__index = function(self, index)
            if index == "s" then return Heartbeat end
            return rawget(OldIndex, index)
        end
        print('AC bypassed')
    else
        print("AC already bypassed")
    end
end

if not game:IsLoaded() then game.Loaded:Wait() end

local function __main()
    local function httpGet(url)
        local ok, res = pcall(function()
            return game:HttpGet(url)
        end)
        if ok and type(res) == "string" and #res > 0 then
            return res
        end
        local req = (syn and syn.request) or (http and http.request) or http_request or request
        if type(req) == "function" then
            local ok2, resp = pcall(req, { Url = url, Method = "GET" })
            if ok2 and type(resp) == "table" then
                local body = resp.Body or resp.body
                if type(body) == "string" and #body > 0 then
                    return body
                end
            end
        end
        return nil
    end

    local function loadFromUrl(url)
        local src = httpGet(url)
        if not src then
            error("HttpGet failed: " .. tostring(url))
        end
        local fn, err = loadstring(src)
        if not fn then
            error("loadstring failed: " .. tostring(err) .. " | " .. tostring(url))
        end
        local ok, result = pcall(fn)
        if not ok then
            error("module error: " .. tostring(result) .. " | " .. tostring(url))
        end
        return result
    end

    local repo = "https://raw.githubusercontent.com/deividcomsono/Obsidian/main/"
    local Library = loadFromUrl(repo .. "Library.lua")
    local ThemeManager = loadFromUrl(repo .. "addons/ThemeManager.lua")
    local SaveManager = loadFromUrl(repo .. "addons/SaveManager.lua")
    local Options = Library.Options
    local Toggles = Library.Toggles
    local newcclosure = newcclosure or function(f) return f end


    local Svc = {
        Run      = game:GetService("RunService"),
        Players  = game:GetService("Players"),
        UIS      = game:GetService("UserInputService"),
        RS       = game:GetService("ReplicatedStorage"),
        Http     = game:GetService("HttpService"),
        WS       = game:GetService("Workspace"),
        RF       = game:GetService("ReplicatedFirst"),
    }

    local M = {
        round = math.round, abs = math.abs, huge = math.huge,
        sqrt  = math.sqrt,  max = math.max, atan2 = math.atan2,
        cos   = math.cos,   sin = math.sin, clamp = math.clamp,
        pi    = math.pi,    min = math.min, floor = math.floor,
    }
    local sfmt  = string.format
    local tIns  = table.insert
    local tClr  = table.clear
    local tFind = table.find
    local tSort = table.sort

    local Camera      = Svc.WS.CurrentCamera
    local LocalPlayer = Svc.Players.LocalPlayer
    local Mouse       = LocalPlayer:GetMouse()
    local PlayerList  = require(Svc.RS:WaitForChild("CustomCharacter"):WaitForChild("PlayerList"))
    local CustomMeshCharacter = require(Svc.RF:WaitForChild("GunSystemPlugins"):WaitForChild("CustomMeshCharacter"))

    local Utils     = {}
    local Cache     = {}
    local Constants = {}

    local State = {
        Config = {
            PlayerNameESP = true, DistanceESP = true, WeaponESP = true,
            HeadESP = true, BoxesESP = true, SkeletonESP = true, TracerESP = false,
            TeamCheck = true, RenderDistance = 10000, VisualFontSize = 19,
            PlayerNameColor = Color3.fromRGB(255,255,255), PlayerNameOutline = true,
            PlayerNameOutlineColor = Color3.fromRGB(0,0,0),
            DistanceColor   = Color3.fromRGB(255,255,255), DistanceOutline = true,
            DistanceOutlineColor   = Color3.fromRGB(0,0,0),
            WeaponColor     = Color3.fromRGB(0,170,255),   WeaponOutline = true,
            WeaponOutlineColor     = Color3.fromRGB(0,0,0),
            HeadColor = Color3.fromRGB(255,0,0), HeadFill = false,
            BoxesColor = Color3.fromRGB(0,255,140), BoxesFill = false, BoxesTransparency = 5,
            SkeletonColor = Color3.fromRGB(255,255,255), TracerColor = Color3.fromRGB(255,255,255),
            NoRecoilEnabled = false,
        },
        VisConfig = {
            FontSize = 19, PosicaoVertical = 0.70, DistanciaMetros = 0.38,
            TextoSemAlvo = "NO TARGET", CorSemAlvo = Color3.fromRGB(150,150,150),
        },
        ESP = {
            Enabled = true,
            PlayersESP = {},
        },
        VisChecker = {
            Enabled = true, cachedIgnore = {}, lastCacheTime = 0,
            CACHE_INTERVAL = 5, FOV_SIZE = 250,
        },
        SpeedHack = {
            Enabled = false, Multiplier = 25, conn = nil,
        },
        Aimbot = {
            Enabled = false,
            Mode = "Silent",
            MouseKey = Enum.UserInputType.MouseButton2,
            NoSpread = false,
            InstantBullet = false,
            ScoutPred = false,
            ResolveY = false,
            PredictionMode = "axal",
            FovSize = 100,
            AimPart = "Head",
            TeamCheck = true,
            ShowFov = true,
            FovColor = Color3.fromRGB(255, 0, 0),
            AimHelicopter = false,
        },
        GunBox = {
            Enabled = true, Width = 280, LineSpacing = 20,
            dragging = false, startPos = nil, mouseStart = nil,
            square = nil, lines = {},
            _lastTarget = nil, _cachedLines = nil,
        },
        Connections = {
            players = nil,
        },
    }

    Constants.HEAD_OFFSET  = Vector3.new(0,  0.9, 0)
    Constants.FEET_OFFSET  = Vector3.new(0, -3.1, 0)
    Constants.V3_ZERO      = Vector3.zero
    Constants.SKEL_PAIRS   = {
        "Head","UpperTorso","UpperTorso","LowerTorso",
        "UpperTorso","LeftUpperArm","UpperTorso","RightUpperArm",
        "LeftUpperArm","LeftLowerArm","RightUpperArm","RightLowerArm",
        "LeftLowerArm","LeftHand","RightLowerArm","RightHand",
        "LowerTorso","LeftUpperLeg","LowerTorso","RightUpperLeg",
        "LeftUpperLeg","LeftLowerLeg","RightUpperLeg","RightLowerLeg",
        "LeftLowerLeg","LeftFoot","RightLowerLeg","RightFoot",
    }

    Cache.playerId   = {}
    Cache.equipName  = {}
    Cache.playerName = {}
    Cache.squad      = {}

    function Utils.getPlayerId(plr)
        local id = Cache.playerId[plr]
        if not id then id = PlayerList:GetUserId(plr); Cache.playerId[plr] = id end
        return id
    end
    Svc.Players.PlayerRemoving:Connect(function(p) Cache.playerId[p] = nil end)

    function Utils.get_safe_character(plr)
        if not plr then return nil end
        local char = PlayerList:GetCharacterFromPlayer(plr)
        if char then
            local worldModel = CustomMeshCharacter:GetWorldCharacterFromPlayer(plr)
            if worldModel then return worldModel end
        end
        if internal_player_table then
            local data = rawget(internal_player_table, Utils.getPlayerId(plr))
            return data and data.WorldModel
        end
        return nil
    end

    function Utils.cleanEquipmentName(val)
        if not val or val == "" then return "None" end
        local cached = Cache.equipName[val]
        if cached then return cached end
        local ok, data = pcall(Svc.Http.JSONDecode, Svc.Http, val)
        local result = (ok and data and data.ClassName) and data.ClassName:gsub("%.[iI]tem$", "") or val
        Cache.equipName[val] = result
        return result
    end

    function Utils.getPlayerNameUpper(plr)
        local n = Cache.playerName[plr]
        if not n then n = plr.Name:upper(); Cache.playerName[plr] = n end
        return n
    end
    Svc.Players.PlayerRemoving:Connect(function(p) Cache.playerName[p] = nil end)

    function Utils.getSquad(plr)
        local s = Cache.squad[plr]
        if s == nil then
            s = plr:GetAttribute("SquadName") or ""
            Cache.squad[plr] = s
            plr:GetAttributeChangedSignal("SquadName"):Connect(function()
                Cache.squad[plr] = plr:GetAttribute("SquadName") or ""
            end)
        end
        return s
    end
    Svc.Players.PlayerRemoving:Connect(function(p) Cache.squad[p] = nil end)

    local internal_player_table
    do
        local custom_mesh_character
        for _, func in {getloadedmodules, getnilinstances} do
            if type(func) ~= "function" then continue end
            for _, mod in func() do
                if mod.Name == "CustomMeshCharacter" then
                    custom_mesh_character = require(mod); break
                end
            end
            if custom_mesh_character then break end
        end
        if custom_mesh_character then
            local ups = debug.getupvalues(custom_mesh_character.GetCharacters)
            for _, v in ups do
                if type(v) == "table" and not v.Player then
                    internal_player_table = v; break
                end
            end
        end
    end

    local entitylist
    for _, gc in getgc(true) do
        if type(gc) == "table" then
            local fn = rawget(gc, "GetPlayerFromWorldCharacter")
            if type(fn) == "function" then
                local getups = getupvalues or debug.getupvalues
                local up    = getups(fn)
                local chars = up[2] and up[2].GetCharacters
                if chars then entitylist = getups(chars)[1]; break end
            end
        end
    end

    local function RemoveESP(plr)
        local drawings = State.ESP.PlayersESP[plr]
        if not drawings then return end
        for _, obj in drawings do
            if type(obj) == "table" then
                for _, l in obj do pcall(l.Remove, l) end
            else
                pcall(obj.Remove, obj)
            end
        end
        State.ESP.PlayersESP[plr] = nil
    end

    local function CreateESP()
        local esp = {}
        local function txt(size, color, outline, outlineColor)
            local t = Drawing.new("Text")
            t.Font = 0; t.Size = size; t.Color = color
            t.Outline = outline; t.OutlineColor = outlineColor; t.Center = true
            return t
        end
        local C   = State.Config
        local fs  = C.VisualFontSize
        esp.Name       = txt(fs,       C.PlayerNameColor, C.PlayerNameOutline, C.PlayerNameOutlineColor)
        esp.Distance   = txt(fs * 0.9, C.DistanceColor,   C.DistanceOutline,   C.DistanceOutlineColor)
        esp.WeaponName = txt(fs * 0.8, C.WeaponColor,     C.WeaponOutline,     C.WeaponOutlineColor)
        local c = Drawing.new("Circle"); c.Thickness = 1; c.Color = C.HeadColor; c.Filled = C.HeadFill
        esp.Head = c
        local s = Drawing.new("Square"); s.Thickness = 1; s.Color = C.BoxesColor; s.Filled = C.BoxesFill
        s.Transparency = C.BoxesTransparency * 0.1; esp.Box = s
        local l = Drawing.new("Line"); l.Thickness = 1; l.Color = C.TracerColor; esp.Tracer = l
        esp.Skeleton = {}
        for _ = 1, 14 do
            local sl = Drawing.new("Line"); sl.Thickness = 1; sl.Color = C.SkeletonColor
            tIns(esp.Skeleton, sl)
        end
        return esp
    end

    local function RefreshAllESP()
        local C   = State.Config
        local nc  = C.PlayerNameColor; local nfs = C.VisualFontSize
        local dc  = C.DistanceColor;   local dfs = nfs * 0.9
        local wc  = C.WeaponColor;     local wfs = nfs * 0.8
        local hc  = C.HeadColor
        local bc  = C.BoxesColor;      local bt  = C.BoxesTransparency * 0.1
        local tc  = C.TracerColor;     local sc  = C.SkeletonColor
        for _, esp in pairs(State.ESP.PlayersESP) do
            if esp.Name       then esp.Name.Color = nc;       esp.Name.Size = nfs end
            if esp.Distance   then esp.Distance.Color = dc;   esp.Distance.Size = dfs end
            if esp.WeaponName then esp.WeaponName.Color = wc; esp.WeaponName.Size = wfs end
            if esp.Head       then esp.Head.Color = hc end
            if esp.Box        then esp.Box.Color = bc;        esp.Box.Transparency = bt end
            if esp.Tracer     then esp.Tracer.Color = tc end
            if esp.Skeleton   then for _, sl in esp.Skeleton do sl.Color = sc end end
        end
    end

    local function hideESP(esp)
        for _, obj in esp do
            if type(obj) == "table" then
                for _, l in obj do l.Visible = false end
            elseif obj.Visible ~= nil then
                obj.Visible = false
            end
        end
    end

    local function UpdateSkeletonLine(line, partA, partB)
        if not (line and partA and partB) then line.Visible = false; return end
        local a, va = Camera:WorldToViewportPoint(partA.Position)
        local b, vb = Camera:WorldToViewportPoint(partB.Position)
        if va and vb then
            line.From = Vector2.new(a.X, a.Y)
            line.To   = Vector2.new(b.X, b.Y)
            line.Visible = true
        else
            line.Visible = false
        end
    end

    local _mySquad = ""
    LocalPlayer:GetAttributeChangedSignal("SquadName"):Connect(function()
        _mySquad = LocalPlayer:GetAttribute("SquadName") or ""
    end)
    _mySquad = LocalPlayer:GetAttribute("SquadName") or ""

    local function UpdateESP(plr, char, esp)
        local C    = State.Config
        local root = char:FindFirstChild("HumanoidRootPart")
        local head = char:FindFirstChild("Head")
        if not (root and head) then hideESP(esp); return end

        local rootPos, onScreen = Camera:WorldToViewportPoint(root.Position)
        if not onScreen then hideESP(esp); return end

        local dist = (Camera.CFrame.Position - root.Position).Magnitude
        if dist > C.RenderDistance then hideESP(esp); return end

        local headTopPos = Camera:WorldToViewportPoint(head.Position + Constants.HEAD_OFFSET)
        local feetPos    = Camera:WorldToViewportPoint(root.Position + Constants.FEET_OFFSET)
        local headPos    = Camera:WorldToViewportPoint(head.Position)
        local boxHeight  = M.abs(feetPos.Y - headTopPos.Y)
        local boxWidth   = boxHeight * 0.55
        local bcX        = rootPos.X
        local bcY        = rootPos.Y - boxHeight * 0.12
        local distMeters = dist * 0.38
        local halfH      = boxHeight * 0.5

        local eName = esp.Name
        if eName then
            eName.Text     = Utils.getPlayerNameUpper(plr)
            eName.Position = Vector2.new(bcX, bcY - halfH - 20)
            eName.Visible  = C.PlayerNameESP
        end

        local gunName = "NONE"
        local curr = plr:FindFirstChild("CurrentSelectedObject")
        if curr and curr.Value and curr.Value.Value then
            gunName = curr.Value.Value.Name:upper()
        end
        local eWep = esp.WeaponName
        if eWep then
            eWep.Text     = "[" .. gunName .. "]"
            eWep.Position = Vector2.new(bcX, bcY - halfH - 35)
            eWep.Visible  = C.WeaponESP
        end

        local eDist = esp.Distance
        if eDist then
            eDist.Text     = M.round(distMeters) .. "M"
            eDist.Position = Vector2.new(bcX, bcY + halfH + 5)
            eDist.Visible  = C.DistanceESP
        end

        local eHead = esp.Head
        if eHead then
            eHead.Position = Vector2.new(headPos.X, headPos.Y)
            eHead.Radius   = boxHeight / 8
            eHead.Visible  = C.HeadESP and distMeters < 120
        end

        local eBox = esp.Box
        if eBox then
            eBox.Position = Vector2.new(bcX - boxWidth * 0.5, bcY - halfH)
            eBox.Size     = Vector2.new(boxWidth, boxHeight)
            eBox.Visible  = C.BoxesESP
        end

        local eTracer = esp.Tracer
        if eTracer then
            local vp = Camera.ViewportSize
            eTracer.From    = Vector2.new(vp.X * 0.5, vp.Y)
            eTracer.To      = Vector2.new(bcX, bcY + halfH)
            eTracer.Visible = C.TracerESP
        end

        local eSkel = esp.Skeleton
        if eSkel then
            local showSkel = C.SkeletonESP
            local partCache = {}
            local function getPart(name)
                local p = partCache[name]
                if p == nil then p = char:FindFirstChild(name); partCache[name] = p or false end
                return p or nil
            end
            for i = 1, 14 do
                local line = eSkel[i]
                line.Visible = showSkel
                if showSkel then
                    local ai = i * 2 - 1
                    UpdateSkeletonLine(
                        line,
                        getPart(Constants.SKEL_PAIRS[ai]),
                        getPart(Constants.SKEL_PAIRS[ai + 1])
                    )
                end
            end
        end
    end

    local function onRender()
        if not State.ESP.Enabled then return end
        local mySquad = State.Config.TeamCheck and _mySquad or nil
        for _, plr in PlayerList:GetPlayers() do
            if plr == LocalPlayer then continue end
            if plr:GetAttribute("Dead") then RemoveESP(plr); continue end
            if mySquad and mySquad ~= "" then
                if Utils.getSquad(plr) == mySquad then RemoveESP(plr); continue end
            end
            local char = Utils.get_safe_character(plr)
            if char then
                if not State.ESP.PlayersESP[plr] then
                    State.ESP.PlayersESP[plr] = CreateESP()
                end
                UpdateESP(plr, char, State.ESP.PlayersESP[plr])
            else
                RemoveESP(plr)
            end
        end
    end

    Svc.Players.PlayerRemoving:Connect(RemoveESP)

    local VC     = State.VisChecker
    local VisCfg = State.VisConfig

    local statusText = Drawing.new("Text")
    statusText.Visible  = true; statusText.Center  = true; statusText.Outline = true
    statusText.Font     = 3;    statusText.Size    = VisCfg.FontSize
    statusText.Color    = VisCfg.CorSemAlvo
    statusText.Position = Vector2.new(
        Camera.ViewportSize.X / 2,
        Camera.ViewportSize.Y * VisCfg.PosicaoVertical
    )

    local rayParams = RaycastParams.new()
    rayParams.FilterType  = Enum.RaycastFilterType.Exclude
    rayParams.IgnoreWater = true

    local _transparentParts = {}
    local _transpConn_add, _transpConn_rem = nil, nil
    local VEGETATION_KEYWORDS = {"grass","foliage","bush","leaf","plant","shrub","weed","fern"}

    local function _checkAndAddPart(obj)
        if not (obj:IsA("BasePart") or obj:IsA("MeshPart") or obj:IsA("UnionOperation")) then return end
        if obj.Transparency >= 0.3 then
            _transparentParts[obj] = true
        else
            local nome = obj.Name:lower()
            for _, kw in ipairs(VEGETATION_KEYWORDS) do
                if nome:find(kw, 1, true) then _transparentParts[obj] = true; break end
            end
        end
    end

    task.spawn(function()
        for _, obj in ipairs(Svc.WS:GetDescendants()) do _checkAndAddPart(obj) end
        _transpConn_add = Svc.WS.DescendantAdded:Connect(_checkAndAddPart)
        _transpConn_rem = Svc.WS.DescendantRemoving:Connect(function(obj) _transparentParts[obj] = nil end)
    end)

    local function buildIgnoreCache()
        local cache = {}
        VC.cachedIgnore = cache
        if LocalPlayer.Character then
            tIns(cache, LocalPlayer.Character)
            for _, d in ipairs(LocalPlayer.Character:GetDescendants()) do tIns(cache, d) end
        end
        local fakeLocal = Utils.get_safe_character(LocalPlayer)
        if fakeLocal then
            tIns(cache, fakeLocal)
            for _, d in ipairs(fakeLocal:GetDescendants()) do tIns(cache, d) end
        end
        for obj in pairs(_transparentParts) do
            if obj.Parent then tIns(cache, obj) end
        end
    end

    local function isPartVisible(part, targetModel)
        if not part then return false end
        local now = tick()
        if now - VC.lastCacheTime > VC.CACHE_INTERVAL then
            buildIgnoreCache(); VC.lastCacheTime = now
        end
        local base    = VC.cachedIgnore; local baseLen = #base
        local extra   = {}; local extraLen = 0
        if LocalPlayer.Character then extraLen += 1; extra[extraLen] = LocalPlayer.Character end
        extraLen += 1; extra[extraLen] = targetModel
        local combined = table.move(base, 1, baseLen, 1, table.create(baseLen + extraLen))
        table.move(extra, 1, extraLen, baseLen + 1, combined)
        rayParams.FilterDescendantsInstances = combined
        local camCF    = Camera.CFrame
        local origin   = camCF.Position + camCF.LookVector * 2
        local toTarget = part.Position - origin
        local dist     = toTarget.Magnitude
        local result   = Svc.WS:Raycast(origin, toTarget.Unit * dist, rayParams)
        if not result then return true end
        if result.Instance == part then return true end
        if result.Instance:IsDescendantOf(targetModel) then return true end
        return false
    end

    local function getNearestCharacterToMouse(mouseLoc)
        local smallest = VC.FOV_SIZE
        local nearPlayer, nearPart, nearModel = nil, nil, nil
        for _, player in PlayerList:GetPlayers() do
            if player == LocalPlayer or player:GetAttribute("Dead") then continue end
            local squad = Utils.getSquad(player)
            if squad ~= "" and squad == _mySquad then continue end
            local fakechar  = PlayerList:GetCharacterFromPlayer(player)
            local character = Utils.get_safe_character(player)
            if not (character and fakechar) then continue end
            local root = fakechar.PrimaryPart or fakechar:FindFirstChild("ServerCollider") or fakechar:FindFirstChild("HumanoidRootPart")
            local head = character:FindFirstChild("Head")
            if not (root and head) then continue end
            local screenPos, onScreen = Camera:WorldToViewportPoint(root.Position)
            if onScreen then
                local dx = screenPos.X - mouseLoc.X; local dy = screenPos.Y - mouseLoc.Y
                local d  = M.sqrt(dx*dx + dy*dy)
                if d < smallest then
                    smallest = d; nearPlayer = player; nearPart = head; nearModel = character
                end
            end
        end
        return nearPlayer, nearPart, nearModel
    end

    local GB = State.GunBox
    GB.square = Drawing.new("Square")
    GB.square.Filled       = true
    GB.square.Transparency = 0.5
    GB.square.Color        = Color3.new(0,0,0)
    GB.square.Visible      = true
    GB.square.ZIndex       = 1
    GB.square.Position     = Vector2.new(Camera.ViewportSize.X - GB.Width - 20, 20)
    GB.square.Size         = Vector2.new(GB.Width, 10)
    for _ = 1, 25 do
        local t = Drawing.new("Text"); t.Size = 16; t.Color = Color3.new(1,1,1)
        t.Outline = true; t.Font = 2; t.Visible = false; t.ZIndex = 2
        tIns(GB.lines, t)
    end

    Svc.UIS.InputBegan:Connect(function(input)
        if not GB.Enabled then return end
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            local mp  = Vector2.new(Mouse.X, Mouse.Y)
            local sq  = GB.square; local sqP = sq.Position; local sqS = sq.Size
            if mp.X >= sqP.X and mp.X <= sqP.X + sqS.X and mp.Y >= sqP.Y and mp.Y <= sqP.Y + sqS.Y then
                GB.dragging = true; GB.startPos = sqP; GB.mouseStart = mp
            end
        end
    end)
    Svc.UIS.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then GB.dragging = false end
    end)
    Svc.UIS.InputChanged:Connect(function(input)
        if GB.dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local delta  = Vector2.new(Mouse.X, Mouse.Y) - GB.mouseStart
            local newPos = GB.startPos + delta
            GB.square.Position = newPos
            for i, t in ipairs(GB.lines) do
                if t.Visible then
                    t.Position = Vector2.new(newPos.X + 5, newPos.Y + 5 + (i-1)*GB.LineSpacing)
                end
            end
        end
    end)

    local function getGunInfoLines(target)
        local lines = GB._cachedLines
        if not lines then lines = {}; GB._cachedLines = lines end
        for i = #lines, 1, -1 do lines[i] = nil end
        if not target or not target:FindFirstChild("GunInventory") then
            tIns(lines, "No target"); return lines
        end
        tIns(lines, target.Name)
        local gunObjects = target:FindFirstChild("GunInventory"):GetChildren()
        if #gunObjects == 0 then
            tIns(lines, "No gun")
        else
            for _, gunObj in ipairs(gunObjects) do
                if gunObj:IsA("ObjectValue") and gunObj.Value then
                    local scopeText  = "No scope"
                    local reticleObj = gunObj:FindFirstChild("AttachmentReticle")
                    if reticleObj and reticleObj:IsA("ObjectValue") and reticleObj.Value then
                        scopeText = tostring(reticleObj.Value) .. "x"
                    end
                    tIns(lines, tostring(gunObj.Value) .. " - " .. scopeText)
                    local mag = gunObj:FindFirstChild("BulletsInMagazine")
                    local res = gunObj:FindFirstChild("BulletsInReserve")
                    if mag and res and mag:IsA("IntValue") and res:IsA("IntValue") then
                        tIns(lines, mag.Value .. " / " .. res.Value)
                    end
                end
            end
        end
        tIns(lines, " ")
        tIns(lines, "Helmet: "   .. Utils.cleanEquipmentName(target:GetAttribute("EquipmentHat")))
        tIns(lines, "Vest: "     .. Utils.cleanEquipmentName(target:GetAttribute("EquipmentVest")))
        tIns(lines, "Backpack: " .. Utils.cleanEquipmentName(target:GetAttribute("EquipmentBackpack")))
        return lines
    end

    local function updateGunBox(mouseLoc)
        if not GB.Enabled then
            GB.square.Visible = false
            for _, t in ipairs(GB.lines) do t.Visible = false end
            return
        end
        GB.square.Visible = true
        local closestPlayer = getNearestCharacterToMouse(mouseLoc)
        local lines = getGunInfoLines(closestPlayer)
        local n     = #lines
        GB.square.Size = Vector2.new(GB.Width, n * GB.LineSpacing + 10)
        local sqPosX  = GB.square.Position.X + 5
        local sqPosY  = GB.square.Position.Y + 5
        local spacing = GB.LineSpacing
        for i = 1, 25 do
            local t = GB.lines[i]
            if i <= n then
                t.Text = lines[i]; t.Position = Vector2.new(sqPosX, sqPosY + (i-1)*spacing); t.Visible = true
            else
                t.Visible = false
            end
        end
    end

    local RunService       = Svc.Run
    local UserInputService = Svc.UIS
    local ReplicatedStorage = Svc.RS

    local gundata   = ReplicatedStorage.GunSystemAssets.GunData
    local sv_config = ReplicatedStorage.CustomCharacterConfigs.Configuration.Server

    local function get_current_gun(plr)
        if not plr then return "Fists" end
        local c = plr:FindFirstChild("CurrentSelectedObject")
        c = c and c.Value; c = c and c.Value
        return c and c.Name or "Fists"
    end

    local function predict_axal(origin, pos, vel, speed, drop)
        local dist = (origin - pos).Magnitude; local t = dist / speed
        local p = pos + vel * t; t = t + (p - pos).Magnitude / speed
        return p + Vector3.yAxis * (drop * t * t)
    end
    local function predict_priv9(origin, pos, vel, speed, drop)
        local t = (origin - pos).Magnitude / speed
        return pos + (vel * t) + Vector3.yAxis * (drop * t * t)
    end
    local function predict_nigger(origin, pos, vel, speed, drop)
        local t = (origin - pos).Magnitude / speed
        return pos + (vel * t) + Vector3.yAxis * (drop * t ^ 2)
    end
    local function predict_nigger_v2(origin, pos, vel, speed, drop)
        local t = (origin - pos).Magnitude / speed
        local predicted = pos + (vel * t)
        t = t + (predicted - pos).Magnitude / speed
        return predicted + Vector3.yAxis * (drop * t ^ 2)
    end

    local function full_prediction(target_position, target_collider)
        if not target_position then return end
        local AB  = State.Aimbot
        local gun = gundata:FindFirstChild(get_current_gun(LocalPlayer))
        local stats = gun and gun:FindFirstChild("Stats")
        local bs    = stats and stats:FindFirstChild("BulletSettings")
        local proj_speed, proj_drop
        if bs then
            local bspeed = bs:FindFirstChild("BulletSpeed"); local bgrav = bs:FindFirstChild("BulletGravity")
            proj_speed = tonumber(bspeed and bspeed.Value) or tonumber(sv_config.sv_default_bullet_speed.Value) or 1500
            proj_drop  = tonumber(bgrav  and bgrav.Value)  or tonumber(sv_config.sv_default_bullet_gravity.Value) or 0
        else
            proj_speed = tonumber(sv_config.sv_default_bullet_speed.Value) or 1500
            proj_drop  = tonumber(sv_config.sv_default_bullet_gravity.Value) or 0
        end
        local campos  = Camera.CFrame.Position
        local velocity = Vector3.zero
        if target_collider then
            if target_collider:IsA("Model") and target_collider.PrimaryPart then
                velocity = target_collider.PrimaryPart.AssemblyLinearVelocity
            elseif target_collider:IsA("BasePart") then
                velocity = target_collider.AssemblyLinearVelocity
            end
        end
        if AB.ResolveY     then velocity = Vector3.new(velocity.X, 0, velocity.Z) end
        if AB.InstantBullet then velocity = Vector3.zero end
        local mode = AB.PredictionMode
        if mode == "axal"       then return predict_axal(campos, target_position, velocity, proj_speed, proj_drop)
        elseif mode == "priv9"  then return predict_priv9(campos, target_position, velocity, proj_speed, proj_drop)
        elseif mode == "nigger" then
            return AB.ScoutPred and predict_nigger(campos, target_position, velocity, proj_speed, proj_drop)
                or predict_axal(campos, target_position, velocity, proj_speed, proj_drop)
        elseif mode == "nigger_v2" then
            return AB.ScoutPred and predict_nigger_v2(campos, target_position, velocity, proj_speed, proj_drop)
                or predict_axal(campos, target_position, velocity, proj_speed, proj_drop)
        end
        return predict_axal(campos, target_position, velocity, proj_speed, proj_drop)
    end

    local function get_closest_target(fov_size, aimpart, team_check)
        local ermm_part, plr_instance, collider = nil, nil, nil
        local maximum_distance = fov_size
        local mousepos         = UserInputService:GetMouseLocation()
        for userid, v in entitylist do
            local player    = v.Player
            if not (player and player ~= LocalPlayer) then continue end
            local root      = v.RootPart; local worldmodel = v.WorldModel; local character = v.Character
            if not (root and worldmodel and character) then continue end
            if type(player) == "table" then
                player = { Name = character.Name .. " (bot)", DisplayName = character.Name .. " (bot)" }
            end
            local part = worldmodel:FindFirstChild(aimpart)
            if not part then continue end
            local position, onscreen = Camera:WorldToViewportPoint(part.Position)
            local distance = (Vector2.new(position.X, position.Y) - mousepos).Magnitude
            if onscreen and distance <= maximum_distance then
                plr_instance = player; ermm_part = part; collider = root; maximum_distance = distance
            end
        end
        if State.Aimbot.AimHelicopter then
            local heliModel = Svc.WS:FindFirstChild("HeliModel")
            if heliModel and heliModel:IsA("Model") then
                local primary = heliModel.PrimaryPart
                if primary then
                    local position, onscreen = Camera:WorldToViewportPoint(primary.Position)
                    local distance = (Vector2.new(position.X, position.Y) - mousepos).Magnitude
                    if onscreen and distance <= maximum_distance then
                        plr_instance = { Name = "Helicopter", DisplayName = "Helicopter", Team = nil }
                        ermm_part = primary; collider = heliModel; maximum_distance = distance
                    end
                end
            end
        end
        return ermm_part, plr_instance, collider
    end

    local fovcircle = Drawing.new("Circle")
    fovcircle.Visible   = false
    fovcircle.Color     = State.Aimbot.FovColor
    fovcircle.Thickness = 1
    fovcircle.NumSides  = 30
    fovcircle.Radius    = State.Aimbot.FovSize

    Svc.Run.Heartbeat:Connect(function()
        local AB  = State.Aimbot
        local cam = Svc.WS.CurrentCamera
        if cam then fovcircle.Position = Vector2.new(cam.ViewportSize.X / 2, cam.ViewportSize.Y / 2) end
        fovcircle.Visible = AB.Enabled and AB.ShowFov
        fovcircle.Radius  = AB.FovSize
        fovcircle.Color   = AB.FovColor
    end)

    local old_buffer_create
    if type(hookfunction) == "function" and type(buffer) == "table" and type(buffer.create) == "function" then
    old_buffer_create = hookfunction(buffer.create, newcclosure(function(size, ...)
        if size ~= 300 then return old_buffer_create(size, ...) end
        if not debug.traceback():find("GunController") then return old_buffer_create(size, ...) end
        local stack = debug.getstack(3, 1)
        if type(stack) ~= "table" then return old_buffer_create(size, ...) end
        if type(stack[3]) == "table" and stack[3].Resimulation ~= nil then return old_buffer_create(size, ...) end
        local AB = State.Aimbot
        if not (AB.Enabled and AB.Mode == "Silent") then return old_buffer_create(size, ...) end
        local part, _, collider = get_closest_target(AB.FovSize, AB.AimPart, AB.TeamCheck)
        local pred
        if part then pred = full_prediction(part.Position, collider) end
        local ld
        if pred then ld = CFrame.lookAt(Camera.CFrame.Position, pred) else ld = Camera.CFrame.LookVector end
        local spread = Vector3.zero
        if AB.NoSpread then
            local rng = Random.new(stack[48] + 1)
            spread = Vector3.new(
                rng:NextNumber() - rng:NextNumber(),
                rng:NextNumber() - rng:NextNumber(),
                rng:NextNumber() - rng:NextNumber()
            ) / stack[22]
        end
        if typeof(ld) == "Vector3" then ld = (ld - spread).Unit else ld = (ld.LookVector - spread).Unit end
        local cf = CFrame.lookAt(Vector3.zero, ld)
        local pitch2, yaw2, _ = cf:ToEulerAnglesYXZ()
        local dir = cf.LookVector
        local r00, r01, r02, r10, r11, r12, r20, r21, r22 = cf:GetComponents()
        stack[32] = cf; stack[33] = dir; stack[34] = dir; stack[36] = pitch2; stack[37] = yaw2
        stack[38] = CFrame.new(0,0,0, r00,r01,r02, r10,r11,r12, r20,r21,r22)
        stack[39] = CFrame.new(0,0,0, r00,r01,r02, r10,r11,r12, r20,r21,r22)
        stack[44] = CFrame.new(0,0,0, r00,r01,r02, r10,r11,r12, r20,r21,r22)
        stack[45] = dir; stack[46] = dir
        pcall(debug.setstack, 3, 1, stack)
        return old_buffer_create(size, ...)
    end))
    else
        old_buffer_create = buffer.create
    end

    local mouse_aim_connection = nil
    local function start_mouse_aim()
        if mouse_aim_connection then return end
        mouse_aim_connection = Svc.Run.Heartbeat:Connect(function()
            local AB = State.Aimbot
            if not (AB.Enabled and AB.Mode == "Mouse") then
                mouse_aim_connection:Disconnect(); mouse_aim_connection = nil; return
            end
            if not UserInputService:IsMouseButtonPressed(AB.MouseKey) then return end
            local part, _, collider = get_closest_target(AB.FovSize, AB.AimPart, AB.TeamCheck)
            if part and collider then
                local pred = full_prediction(part.Position, collider)
                if pred then
                    local pos  = Camera:WorldToViewportPoint(pred)
                    local mpos = UserInputService:GetMouseLocation()
                    local diff = Vector2.new(pos.X - mpos.X, pos.Y - mpos.Y)
                    if diff.Magnitude > 1 then mousemoverel(diff.X, diff.Y) end
                end
            end
        end)
    end

    local function set_aimbot_enabled(state)
        State.Aimbot.Enabled = state
        if not state then
            if mouse_aim_connection then mouse_aim_connection:Disconnect(); mouse_aim_connection = nil end
        else
            if State.Aimbot.Mode == "Mouse" then start_mouse_aim() end
        end
    end
    local function set_aimbot_mode(mode)
        State.Aimbot.Mode = mode
        if State.Aimbot.Enabled then
            if mode == "Mouse" then start_mouse_aim()
            else
                if mouse_aim_connection then mouse_aim_connection:Disconnect(); mouse_aim_connection = nil end
            end
        end
    end

    local function ApplyNoRecoil()
        if rawget(_G, "NoRecoilHooked") then return end
        for _, gc in getgc(true) do
            if type(gc) == "table" and rawget(gc, "Impulse") and rawget(gc, "SetPosition") then
                local oldImpulse = gc.Impulse; local oldSetPos = gc.SetPosition
                gc.Impulse     = function(self,...) if State.Config.NoRecoilEnabled then return end return oldImpulse(self,...) end
                gc.SetPosition = function(self,...) if State.Config.NoRecoilEnabled then return end return oldSetPos(self,...) end
            end
        end
        rawset(_G, "NoRecoilHooked", true)
    end
    task.spawn(ApplyNoRecoil)

    local function GetMovementPart()
        for _, child in Camera:GetDescendants() do
            if child:IsA("MeshPart") then
                local sz = child.Size
                if sz.X == 2.5 and sz.Z == 2.5 and (sz.Y == 5 or sz.Y == 3.25 or sz.Y == 3 or sz.Y == 2) then
                    return child
                end
            end
        end
        return nil
    end

    local function ApplySpeedHack()
        local SH = State.SpeedHack
        if not SH.Enabled then return end
        local MovementPart = GetMovementPart()
        if not MovementPart then return end
        local dx, dz = 0, 0
        local lv = Camera.CFrame.LookVector; local rv = Camera.CFrame.RightVector
        if Svc.UIS:IsKeyDown(Enum.KeyCode.W) then dx += lv.X; dz += lv.Z end
        if Svc.UIS:IsKeyDown(Enum.KeyCode.S) then dx -= lv.X; dz -= lv.Z end
        if Svc.UIS:IsKeyDown(Enum.KeyCode.A) then dx -= rv.X; dz -= rv.Z end
        if Svc.UIS:IsKeyDown(Enum.KeyCode.D) then dx += rv.X; dz += rv.Z end
        if dx ~= 0 or dz ~= 0 then
            local mul = SH.Multiplier / M.sqrt(dx*dx + dz*dz)
            pcall(function()
                MovementPart.AssemblyLinearVelocity = Vector3.new(
                    dx*mul, MovementPart.AssemblyLinearVelocity.Y, dz*mul)
            end)
        end
    end

    local function ToggleSpeedHack(value)
        local SH = State.SpeedHack
        SH.Enabled = value
        if value then
            if not SH.conn then SH.conn = Svc.Run.Heartbeat:Connect(ApplySpeedHack) end
        else
            if SH.conn then SH.conn:Disconnect(); SH.conn = nil end
        end
    end


    local Conn = State.Connections
    local function startPlayerESP()
        if not Conn.players then Conn.players = Svc.Run.RenderStepped:Connect(onRender) end
    end
    local function stopPlayerESP()
        if Conn.players then Conn.players:Disconnect(); Conn.players = nil end
        for plr in pairs(State.ESP.PlayersESP) do RemoveESP(plr) end
        State.ESP.PlayersESP = {}
    end


    Svc.Run.RenderStepped:Connect(function()
        local VisCfg_ = State.VisConfig
        local mouseLoc = Svc.UIS:GetMouseLocation()
        local vp       = Camera.ViewportSize
        local centerX  = vp.X * 0.5; local centerY = vp.Y * 0.5

        statusText.Size     = VisCfg_.FontSize
        statusText.Position = Vector2.new(centerX, vp.Y * VisCfg_.PosicaoVertical)

        if not VC.Enabled then
            statusText.Visible = false
        else
            statusText.Visible = true
            local targetPlayer, targetHead, targetModel = getNearestCharacterToMouse(mouseLoc)
            if targetPlayer and targetHead then
                local visible    = isPartVisible(targetHead, targetModel)
                local distMeters = (targetHead.Position - Camera.CFrame.Position).Magnitude * VisCfg_.DistanciaMetros
                local squad      = Utils.getSquad(targetPlayer)
                local squadText  = (squad and squad ~= "") and ("Squad: " .. squad) or ""
                statusText.Text  = string.format("%s [%.0fM] | %s\n%s",
                    Utils.getPlayerNameUpper(targetPlayer), distMeters,
                    visible and "VISIBLE" or "NOT VISIBLE", squadText)
                statusText.Color = visible and Color3.fromRGB(0,255,0) or Color3.fromRGB(255,0,0)
            else
                statusText.Text  = VisCfg_.TextoSemAlvo
                statusText.Color = VisCfg_.CorSemAlvo
            end
        end

        updateGunBox(mouseLoc)
    end)

    local function setupUI()
        local Window = Library:CreateWindow({Title="Nova Menu", Footer="Made with AI by Akko", ShowCustomCursor=true, NotifySide="Right"})
        local Tabs = {
            Combat   = Window:AddTab("Combat",   "crosshair"),
            ESP      = Window:AddTab("ESP",      "scan-eye"),
            Exploit  = Window:AddTab("Exploit",  "skull"),
            Settings = Window:AddTab("Settings", "settings"),
        }

        do
            local AB = State.Aimbot
            local AimbotGroup = Tabs.Combat:AddLeftGroupbox("Aimbot")
            AimbotGroup:AddToggle("AimbotEnabled",{Text="Enable Aimbot",Default=false,Callback=function(v) set_aimbot_enabled(v) end})
                :AddKeyPicker("AimbotKeybind",{Default="F",NoUI=false,Text="Toggle Aimbot",Mode="Toggle"})
            Options.AimbotKeybind:OnClick(function() Toggles.AimbotEnabled:SetValue(not Toggles.AimbotEnabled.Value) end)
            AimbotGroup:AddToggle("AimHelicopter",{Text="Helicopter Aimbot",Default=false,Callback=function(v) AB.AimHelicopter = v end})
            AimbotGroup:AddDropdown("AimbotMode",{Values={"Silent","Mouse"},Default="Silent",Text="Aim Mode",Callback=function(v) set_aimbot_mode(v) end})
            AimbotGroup:AddDropdown("MouseKey",{Values={"MouseButton1","MouseButton2","MouseButton3"},Default="MouseButton2",Text="Mouse Key (Hold)",Callback=function(v) AB.MouseKey = Enum.UserInputType[v] end})
            AimbotGroup:AddToggle("ShowFov",{Text="Show FOV Circle",Default=true,Callback=function(v) AB.ShowFov = v end})
                :AddColorPicker("FovColor",{Default=Color3.fromRGB(255,0,0),Title="FOV Color",Callback=function(c) AB.FovColor = c end})
            AimbotGroup:AddSlider("FovSize",{Text="FOV Size",Default=100,Min=10,Max=500,Rounding=0,Callback=function(v) AB.FovSize = v end})
            AimbotGroup:AddDropdown("AimPart",{Values={"Head","UpperTorso","LowerTorso","LeftUpperArm","RightUpperArm","LeftLowerArm","RightLowerArm"},Default="Head",Text="Target Part",Callback=function(v) AB.AimPart = v end})
            AimbotGroup:AddToggle("NoSpread",{Text="No Spread",Default=false,Callback=function(v) AB.NoSpread = v end})
            AimbotGroup:AddToggle("InstantBullet",{Text="Instant Bullet",Default=false,Callback=function(v) AB.InstantBullet = v end})
            AimbotGroup:AddToggle("ResolveY",{Text="Resolve Y Velocity",Default=false,Callback=function(v) AB.ResolveY = v end})
            AimbotGroup:AddToggle("ScoutPred",{Text="Scout Prediction",Default=false,Callback=function(v) AB.ScoutPred = v end})
            AimbotGroup:AddDropdown("PredictionMode",{Values={"axal","priv9","nigger","nigger_v2"},Default="axal",Text="Prediction Type",Callback=function(v) AB.PredictionMode = v end})
            AimbotGroup:AddToggle("TeamCheck",{Text="Team Check",Default=true,Callback=function(v) AB.TeamCheck = v end})
        end

        do
            local NRGroup = Tabs.Combat:AddRightGroupbox("No Recoil")
            NRGroup:AddToggle("NoRecoilEnabled",{Text="No Recoil",Default=false})
                :AddKeyPicker("NoRecoilKey",{Default="N",NoUI=false,Text="Toggle No Recoil",Mode="Toggle"})
            Toggles.NoRecoilEnabled:OnChanged(function(v) State.Config.NoRecoilEnabled = v end)
            Options.NoRecoilKey:OnClick(function() Toggles.NoRecoilEnabled:SetValue(not Toggles.NoRecoilEnabled.Value) end)
        end

        do
            local C      = State.Config
            local ESPLeft = Tabs.ESP:AddLeftGroupbox("Player ESP")
            ESPLeft:AddToggle("ESPEnabled",{Text="Enable Player ESP",Default=true})
                :AddKeyPicker("ESPKeybind",{Default="F1",NoUI=false,Text="Toggle Player ESP",Mode="Toggle"})
            Toggles.ESPEnabled:OnChanged(function()
                State.ESP.Enabled = Toggles.ESPEnabled.Value
                if State.ESP.Enabled then startPlayerESP() else stopPlayerESP() end
            end)
            Options.ESPKeybind:OnClick(function() Toggles.ESPEnabled:SetValue(not Toggles.ESPEnabled.Value) end)
            ESPLeft:AddToggle("ESPTeamCheck",{Text="Team Check",Default=true})
            Toggles.ESPTeamCheck:OnChanged(function() C.TeamCheck = Toggles.ESPTeamCheck.Value end)
            ESPLeft:AddToggle("NameESP",{Text="Name",Default=true}):AddColorPicker("NameColor",{Default=Color3.fromRGB(255,255,255),Title="Name Color"})
            Toggles.NameESP:OnChanged(function() C.PlayerNameESP = Toggles.NameESP.Value end)
            Options.NameColor:OnChanged(function() C.PlayerNameColor = Options.NameColor.Value; RefreshAllESP() end)
            ESPLeft:AddToggle("DistESP",{Text="Distance",Default=true}):AddColorPicker("DistColor",{Default=Color3.fromRGB(255,255,255),Title="Distance Color"})
            Toggles.DistESP:OnChanged(function() C.DistanceESP = Toggles.DistESP.Value end)
            Options.DistColor:OnChanged(function() C.DistanceColor = Options.DistColor.Value; RefreshAllESP() end)
            ESPLeft:AddToggle("WeaponESP",{Text="Weapon",Default=true}):AddColorPicker("WeaponColor",{Default=Color3.fromRGB(0,170,255),Title="Weapon Color"})
            Toggles.WeaponESP:OnChanged(function() C.WeaponESP = Toggles.WeaponESP.Value end)
            Options.WeaponColor:OnChanged(function() C.WeaponColor = Options.WeaponColor.Value; RefreshAllESP() end)
            ESPLeft:AddToggle("HeadESP",{Text="Head Circle",Default=true}):AddColorPicker("HeadColor",{Default=Color3.fromRGB(255,0,0),Title="Head Color"})
            Toggles.HeadESP:OnChanged(function() C.HeadESP = Toggles.HeadESP.Value end)
            Options.HeadColor:OnChanged(function() C.HeadColor = Options.HeadColor.Value; RefreshAllESP() end)
            ESPLeft:AddToggle("BoxESP",{Text="Boxes",Default=true}):AddColorPicker("BoxColor",{Default=Color3.fromRGB(0,255,140),Title="Box Color"})
            Toggles.BoxESP:OnChanged(function() C.BoxesESP = Toggles.BoxESP.Value end)
            Options.BoxColor:OnChanged(function() C.BoxesColor = Options.BoxColor.Value; RefreshAllESP() end)
            ESPLeft:AddToggle("SkelESP",{Text="Skeleton",Default=true}):AddColorPicker("SkelColor",{Default=Color3.fromRGB(255,255,255),Title="Skeleton Color"})
            Toggles.SkelESP:OnChanged(function() C.SkeletonESP = Toggles.SkelESP.Value end)
            Options.SkelColor:OnChanged(function() C.SkeletonColor = Options.SkelColor.Value; RefreshAllESP() end)
            ESPLeft:AddToggle("TracerESP",{Text="Tracer",Default=false}):AddColorPicker("TracerColor",{Default=Color3.fromRGB(255,255,255),Title="Tracer Color"})
            Toggles.TracerESP:OnChanged(function() C.TracerESP = Toggles.TracerESP.Value end)
            Options.TracerColor:OnChanged(function() C.TracerColor = Options.TracerColor.Value; RefreshAllESP() end)
            ESPLeft:AddSlider("RenderDist",{Text="Render Distance",Default=10000,Min=100,Max=20000,Rounding=0})
            Options.RenderDist:OnChanged(function() C.RenderDistance = Options.RenderDist.Value end)
            ESPLeft:AddSlider("FontSize",{Text="Font Size",Default=19,Min=10,Max=30,Rounding=0})
            Options.FontSize:OnChanged(function() C.VisualFontSize = Options.FontSize.Value; RefreshAllESP() end)
        end

        do
            local VisRight = Tabs.ESP:AddRightGroupbox("Visible Checker")
            VisRight:AddToggle("VisChecker",{Text="Enable Visible Checker",Default=true})
                :AddKeyPicker("VisKeybind",{Default="F4",NoUI=false,Text="Toggle Vis Checker",Mode="Toggle"})
            Toggles.VisChecker:OnChanged(function() VC.Enabled = Toggles.VisChecker.Value end)
            Options.VisKeybind:OnClick(function() Toggles.VisChecker:SetValue(not Toggles.VisChecker.Value) end)
            VisRight:AddSlider("VisFontSize",{Text="Font Size",Default=19,Min=10,Max=30,Rounding=0})
            Options.VisFontSize:OnChanged(function() State.VisConfig.FontSize = Options.VisFontSize.Value end)
            VisRight:AddSlider("VisPosY",{Text="Vertical Position",Default=70,Min=10,Max=100,Rounding=0})
            Options.VisPosY:OnChanged(function() State.VisConfig.PosicaoVertical = Options.VisPosY.Value / 100 end)
        end

        do
            local GunGroup = Tabs.ESP:AddRightGroupbox("Inventory View")
            GunGroup:AddToggle("GunBox",{Text="Gun Info Box",Default=true})
                :AddKeyPicker("GunBoxKeybind",{Default="F5",NoUI=false,Text="Toggle Gun Box",Mode="Toggle"})
            Toggles.GunBox:OnChanged(function() State.GunBox.Enabled = Toggles.GunBox.Value end)
            Options.GunBoxKeybind:OnClick(function() Toggles.GunBox:SetValue(not Toggles.GunBox.Value) end)
        end

       do
    local SH = State.SpeedHack
    local InputManager = require(Svc.RS:WaitForChild("CustomCharacter"):WaitForChild("InputManager"))
    local JumpBind     = InputManager:GetKeybindByName("Jump")
    if JumpBind then JumpBind:SetActivationType("HoldRepeat") end

    local InfJumpHook = nil
    local function ApplyInfJump()
        if InfJumpHook then return end
        for _, v in getgc() do
            if type(v) == "function" then
                local info = debug.getinfo(v)
                if info.name == "updateCharData" then
                    local old; old = hookfunction(v, newcclosure(function(p254, p255, p256)
                        if SH.Enabled and p254 == 'Jump' then return end
                        return old(p254, p255, p256)
                    end))
                    InfJumpHook = old; break
                end
            end
        end
    end
    task.spawn(ApplyInfJump)

    local function ApplySpeedHack()
        if not SH.Enabled then return end
        local MovementPart = GetMovementPart()
        if not MovementPart then return end
        local dx, dz = 0, 0
        local lv = Camera.CFrame.LookVector; local rv = Camera.CFrame.RightVector
        if Svc.UIS:IsKeyDown(Enum.KeyCode.W) then dx += lv.X; dz += lv.Z end
        if Svc.UIS:IsKeyDown(Enum.KeyCode.S) then dx -= lv.X; dz -= lv.Z end
        if Svc.UIS:IsKeyDown(Enum.KeyCode.A) then dx -= rv.X; dz -= rv.Z end
        if Svc.UIS:IsKeyDown(Enum.KeyCode.D) then dx += rv.X; dz += rv.Z end
        if dx ~= 0 or dz ~= 0 then
            local mul = SH.Multiplier / M.sqrt(dx*dx + dz*dz)
            pcall(function()
                MovementPart.AssemblyLinearVelocity = Vector3.new(
                    dx*mul, MovementPart.AssemblyLinearVelocity.Y, dz*mul)
            end)
        end
    end

    local function ToggleSpeedHack(value)
        SH.Enabled = value
        if value then
            if not SH.conn then SH.conn = Svc.Run.Heartbeat:Connect(ApplySpeedHack) end
        else
            if SH.conn then SH.conn:Disconnect(); SH.conn = nil end
        end
    end

    local SpeedGroup = Tabs.Exploit:AddLeftGroupbox("Speed Hack")
    SpeedGroup:AddToggle("SpeedHackToggle", {Text="Enable Speed Hack", Default=false})
        :AddKeyPicker("SpeedHackKey", {Default="F7", NoUI=false, Text="Toggle Speed Hack", Mode="Toggle"})
    Toggles.SpeedHackToggle:OnChanged(function() ToggleSpeedHack(Toggles.SpeedHackToggle.Value) end)
    Options.SpeedHackKey:OnClick(function() Toggles.SpeedHackToggle:SetValue(not Toggles.SpeedHackToggle.Value) end)
    SpeedGroup:AddSlider("SpeedMultiplier", {Text="Speed Multiplier", Default=25, Min=1, Max=75, Rounding=1, Suffix="x"})
    Options.SpeedMultiplier:OnChanged(function() SH.Multiplier = Options.SpeedMultiplier.Value end)
end

    -- ──────────────────────────────────────────────────────────────
    -- ANTI-AIM
    -- ──────────────────────────────────────────────────────────────
    local AntiAim = {
        Enabled       = false,
        PitchFixed    = -math.pi / 2,
        YawFixed      = math.pi,
        JitterEnabled = false,
        JitterOffset  = math.pi / 3,
        JitterSpeed   = 2.0,
        SpinEnabled   = false,
        SpinSpeed     = 3.0,
        YawEnabled    = false,
        YawMode       = "Forward",
        YawSpinSpeed  = 5,
        _currentPitch = 0,
        _currentYaw   = 0,
        _time         = 0,
        _spinAngle    = 0,
        _yawAngle     = 0,
        _yawSpin      = 0,
        _renderConn   = nil,
        _yawHb        = nil,
        _yawRs        = nil,
        _charPlugin   = nil,
    }

    local AA_old_index = nil

    local function InstallAntiAimHook()
        if AA_old_index then return end
        AA_old_index = hookfunction(
            getrawmetatable(game).__index,
            newcclosure(function(self, key)
                if key ~= "CFrame" or self ~= Camera then
                    return AA_old_index(self, key)
                end
                if checkcaller() then
                    return AA_old_index(self, key)
                end
                local tb = debug.traceback()
                if tb:find("GunController") or tb:find("GunPlugin") then
                    return AA_old_index(self, key)
                end
                if not AntiAim.Enabled then
                    return AA_old_index(self, key)
                end
                local cf = AA_old_index(self, key)
                local p, y = AntiAim._currentPitch, AntiAim._currentYaw
                local dir = Vector3.new(
                    M.sin(y) * M.cos(p),
                    M.sin(p),
                    M.cos(y) * M.cos(p)
                )
                return CFrame.lookAt(cf.Position, cf.Position + dir)
            end)
        )
    end

    local function StartAntiAim()
        if AntiAim._renderConn then return end
        InstallAntiAimHook()
        AntiAim._spinAngle = 0
        AntiAim._renderConn = Svc.Run.RenderStepped:Connect(function(dt)
            AntiAim._time = AntiAim._time + dt
            local cam = Camera
            local camYaw = 0
            if cam then
                local lv = cam.CFrame.LookVector
                camYaw = M.atan2(lv.X, lv.Z)
            end
            local pitch = AntiAim.PitchFixed
            local yaw   = camYaw + AntiAim.YawFixed
            if AntiAim.JitterEnabled then
                local t = AntiAim._time
                pitch = pitch + AntiAim.JitterOffset * M.sin(t * AntiAim.JitterSpeed)
                yaw   = yaw   + AntiAim.JitterOffset * M.sin(t * AntiAim.JitterSpeed + 1.2)
            end
            if AntiAim.SpinEnabled then
                AntiAim._spinAngle = AntiAim._spinAngle + AntiAim.SpinSpeed * dt
                yaw = yaw + AntiAim._spinAngle
            else
                AntiAim._spinAngle = 0
            end
            AntiAim._currentPitch = pitch
            AntiAim._currentYaw   = yaw
        end)
    end

    local function StopAntiAim()
        if AntiAim._renderConn then
            AntiAim._renderConn:Disconnect()
            AntiAim._renderConn = nil
        end
        AntiAim.Enabled       = false
        AntiAim._time         = 0
        AntiAim._currentPitch = 0
        AntiAim._currentYaw   = 0
        AntiAim._spinAngle    = 0
    end

    local function ensureCharPlugin()
        if AntiAim._charPlugin then return AntiAim._charPlugin end
        pcall(function()
            local rf  = Svc.RF
            local mod = rf:WaitForChild("CustomCharacter", 5)
            mod = mod and mod:WaitForChild("CharacterController", 5)
            mod = mod and mod:WaitForChild("CharacterPlugin", 5)
            if mod then AntiAim._charPlugin = require(mod) end
        end)
        return AntiAim._charPlugin
    end

    local function StartYaw()
        if AntiAim._yawHb or AntiAim._yawRs then return end
        task.spawn(ensureCharPlugin)
        AntiAim._yawHb = Svc.Run.Heartbeat:Connect(function(dt)
            if not AntiAim.YawEnabled then return end
            local cam = Camera
            local camYaw = 0
            if cam then
                local lv = cam.CFrame.LookVector
                camYaw = M.atan2(-lv.X, -lv.Z)
            end
            if AntiAim.YawMode == "Spin" then
                local spd = tonumber(AntiAim.YawSpinSpeed) or 5
                AntiAim._yawSpin  = ((AntiAim._yawSpin or 0) + spd * dt) % (2 * math.pi)
                AntiAim._yawAngle = (camYaw + AntiAim._yawSpin) % (2 * math.pi)
            elseif AntiAim.YawMode == "Forward" then
                AntiAim._yawAngle = (camYaw + math.pi) % (2 * math.pi)
            elseif AntiAim.YawMode == "Backward" then
                AntiAim._yawAngle = camYaw % (2 * math.pi)
            end
        end)
        AntiAim._yawRs = Svc.Run.RenderStepped:Connect(function()
            if not AntiAim.YawEnabled then return end
            local angle = AntiAim._yawAngle or 0
            local cc = AntiAim._charPlugin or ensureCharPlugin()
            if cc and cc.SetYaw then
                pcall(function() cc:SetYaw(angle) end)
            end
            if LocalPlayer and LocalPlayer:FindFirstChild("CharData") and LocalPlayer.CharData:FindFirstChild("Yaw") then
                pcall(function() LocalPlayer.CharData.Yaw.Value = angle end)
            end
            if LocalPlayer and LocalPlayer.Character then
                local hrp = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                if hrp then
                    pcall(function()
                        hrp.CFrame = CFrame.new(hrp.Position) * CFrame.Angles(0, angle, 0)
                    end)
                end
            end
        end)
    end

    local function StopYaw()
        if AntiAim._yawHb then AntiAim._yawHb:Disconnect(); AntiAim._yawHb = nil end
        if AntiAim._yawRs then AntiAim._yawRs:Disconnect(); AntiAim._yawRs = nil end
        AntiAim._yawAngle  = 0
        AntiAim._yawSpin   = 0
        AntiAim.YawEnabled = false
    end

    local function ToggleAntiAim(state)
        AntiAim.Enabled = state
        if state then StartAntiAim() else StopAntiAim() end
    end

    local function ToggleYaw(state)
        AntiAim.YawEnabled = state
        if state then StartYaw() else StopYaw() end
    end

    InstallAntiAimHook()

    local AAGroup = Tabs.Exploit:AddLeftGroupbox("Anti-Aim")
    AAGroup:AddToggle("AntiAimEnabled", {Text="Enable Anti-Aim", Default=false})
        :AddKeyPicker("AntiAimKey", {Default="F8", NoUI=false, Text="Toggle Anti-Aim", Mode="Toggle"})
    Toggles.AntiAimEnabled:OnChanged(function() ToggleAntiAim(Toggles.AntiAimEnabled.Value) end)
    Options.AntiAimKey:OnClick(function() Toggles.AntiAimEnabled:SetValue(not Toggles.AntiAimEnabled.Value) end)
    AAGroup:AddSlider("AAPitch", {Text="Pitch", Default=-90, Min=-180, Max=180, Rounding=1, Suffix="°"})
    Options.AAPitch:OnChanged(function() AntiAim.PitchFixed = math.rad(Options.AAPitch.Value) end)
    AAGroup:AddToggle("AAJitter", {Text="Jitter", Default=false})
    Toggles.AAJitter:OnChanged(function() AntiAim.JitterEnabled = Toggles.AAJitter.Value end)
    AAGroup:AddSlider("AAJitterOffset", {Text="Jitter Offset", Default=60, Min=1, Max=180, Rounding=1, Suffix="°"})
    Options.AAJitterOffset:OnChanged(function() AntiAim.JitterOffset = math.rad(Options.AAJitterOffset.Value) end)
    AAGroup:AddSlider("AAJitterSpeed", {Text="Jitter Speed", Default=2, Min=0.5, Max=10, Rounding=1})
    Options.AAJitterSpeed:OnChanged(function() AntiAim.JitterSpeed = Options.AAJitterSpeed.Value end)
    AAGroup:AddToggle("AASpin", {Text="Spin", Default=false})
    Toggles.AASpin:OnChanged(function() AntiAim.SpinEnabled = Toggles.AASpin.Value end)
    AAGroup:AddSlider("AASpinSpeed", {Text="Spin Speed", Default=3, Min=0.5, Max=15, Rounding=1})
    Options.AASpinSpeed:OnChanged(function() AntiAim.SpinSpeed = Options.AASpinSpeed.Value end)

    local AAYawGroup = Tabs.Exploit:AddRightGroupbox("Yaw Control")
    AAYawGroup:AddToggle("YawEnabled", {Text="Enable Yaw", Default=false})
    Toggles.YawEnabled:OnChanged(function() ToggleYaw(Toggles.YawEnabled.Value) end)
    AAYawGroup:AddDropdown("YawMode", {Values={"Forward","Backward","Spin"}, Default="Forward", Text="Yaw Mode"})
    Options.YawMode:OnChanged(function() AntiAim.YawMode = Options.YawMode.Value end)
    AAYawGroup:AddSlider("YawSpinSpeed", {Text="Yaw Spin Speed", Default=5, Min=1, Max=20, Rounding=1})
    Options.YawSpinSpeed:OnChanged(function() AntiAim.YawSpinSpeed = Options.YawSpinSpeed.Value end)

    -- ──────────────────────────────────────────────────────────────
    -- ELEVATE
    -- ──────────────────────────────────────────────────────────────
    local Elevate = { Amount = 8 }

    local function DoElevate()
        local amount = tonumber(Elevate.Amount) or 8
        local char   = LocalPlayer.Character
        if not char then return end
        local target = char:FindFirstChild("HumanoidRootPart") or char.PrimaryPart
        if not target then return end
        local part = Instance.new("Part")
        part.Size         = Vector3.new(6, 0.25, 6)
        part.Anchored     = true
        part.CanCollide   = true
        part.Transparency = 1
        part.CFrame       = CFrame.new(target.Position.X, target.Position.Y - 1.2, target.Position.Z)
        part.Parent       = Svc.WS
        for _ = 1, amount do
            part.CFrame = part.CFrame + Vector3.new(0, 1.15, 0)
            task.wait(0.012)
        end
        task.wait(0.05)
        part:Destroy()
    end

    local ElevGroup = Tabs.Exploit:AddLeftGroupbox("Elevate")
    ElevGroup:AddSlider("ElevateAmount", {Text="Height Steps", Default=8, Min=1, Max=30, Rounding=1})
    Options.ElevateAmount:OnChanged(function() Elevate.Amount = Options.ElevateAmount.Value end)
    ElevGroup:AddButton("Elevate", function() DoElevate() end)

    -- ──────────────────────────────────────────────────────────────
    -- INSTANT AIM
    -- ──────────────────────────────────────────────────────────────
    local InstantAim = {
        Enabled           = false,
        OriginalValues    = {},
        OriginalFunctions = {},
    }

    local _iaTargets = nil

    local function findInstantAimTargets()
        if _iaTargets then return _iaTargets end
        _iaTargets = {}
        for _, v in next, getgc(true) do
            if type(v) == "table" then
                for i, val in next, v do
                    if type(i) == "string" and i:find("GunAim") then
                        if type(val) == "number" or type(val) == "function" then
                            _iaTargets[#_iaTargets + 1] = { tbl = v, key = i, val = val }
                        end
                    end
                end
            end
        end
        return _iaTargets
    end

    local function ToggleInstantAim(state)
        if state then
            if InstantAim.Enabled then return end
            for _, entry in ipairs(findInstantAimTargets()) do
                local v, i, val = entry.tbl, entry.key, entry.val
                if type(val) == "number" then
                    InstantAim.OriginalValues[v] = InstantAim.OriginalValues[v] or {}
                    InstantAim.OriginalValues[v][i] = val
                    v[i] = 100000000
                elseif type(val) == "function" then
                    InstantAim.OriginalFunctions[val] = val
                    if isfunctionhooked(val) then restorefunction(val) end
                    hookfunction(val, function() return 100000000 end)
                end
            end
            InstantAim.Enabled = true
        else
            if not InstantAim.Enabled then return end
            for tbl, keys in pairs(InstantAim.OriginalValues) do
                for k, origVal in pairs(keys) do
                    pcall(function() tbl[k] = origVal end)
                end
            end
            InstantAim.OriginalValues = {}
            for fn in pairs(InstantAim.OriginalFunctions) do
                pcall(restorefunction, fn)
            end
            table.clear(InstantAim.OriginalFunctions)
            InstantAim.Enabled = false
        end
    end

    local IAGroup = Tabs.Exploit:AddRightGroupbox("Instant Aim")
    IAGroup:AddToggle("InstantAimEnabled", {Text="Enable Instant Aim", Default=false})
        :AddKeyPicker("InstantAimKey", {Default="F9", NoUI=false, Text="Toggle Instant Aim", Mode="Toggle"})
    Toggles.InstantAimEnabled:OnChanged(function() ToggleInstantAim(Toggles.InstantAimEnabled.Value) end)
    Options.InstantAimKey:OnClick(function() Toggles.InstantAimEnabled:SetValue(not Toggles.InstantAimEnabled.Value) end)

    -- ──────────────────────────────────────────────────────────────
    -- NPC SCANNER
    -- ──────────────────────────────────────────────────────────────
    local NPCScanner = {
        _emberClient = nil,
        _npcService  = nil,
    }

    local function ensureEmberClient()
        if NPCScanner._emberClient then return NPCScanner._emberClient end
        pcall(function()
            NPCScanner._emberClient = require(
                Svc.RF
                    :WaitForChild("EmberClientLibrary")
                    :WaitForChild("EmberClient")
                    :WaitForChild("EmberClient")
            )
            NPCScanner._npcService = NPCScanner._emberClient:GetService("NPCSimulatorService")
        end)
        return NPCScanner._emberClient
    end

    local function RunNPCScan()
        ensureEmberClient()
        local svc = NPCScanner._npcService
        if not svc then
            warn("[NPCScanner] NPCSimulatorService unavailable")
            return
        end
        local total = svc.TotalNPCs or 0
        print(string.format("[NPCScanner] Scanning %d zombies", total))
        local found = {}
        for _, Zombie in next, svc.NPCs do
            if type(Zombie) == "table" or type(Zombie) == "userdata" then
                local equipment = Zombie.Equipment
                if equipment then
                    for _, Item in next, equipment do
                        local ok, err = pcall(function()
                            local ItemClass = Item.ClassName or ""
                            local Skin      = Item.SkinOverride
                            if ItemClass:find("Altyn") then
                                local label = ItemClass:gsub("%.item", "")
                                print(string.format("[NPCScanner] Chinese zombie detected (%s)", label))
                                table.insert(found, {type="Chinese", label=label})
                            elseif Skin and Skin:find("Beret") then
                                print(string.format("[NPCScanner] Tactical zombie detected (%s)", Skin))
                                table.insert(found, {type="Tactical", label=Skin})
                            end
                        end)
                        if not ok then
                            warn("[NPCScanner] Item read error: " .. tostring(err))
                        end
                    end
                end
            end
        end
        if #found == 0 then
            print("[NPCScanner] No special zombies detected")
        else
            print(string.format("[NPCScanner] Scan complete — %d special zombie(s) found", #found))
        end
        return found
    end

    local NPCGroup = Tabs.Exploit:AddLeftGroupbox("NPC Scanner")
    NPCGroup:AddButton("Scan NPCs", function() task.spawn(RunNPCScan) end)


    -- ──────────────────────────────────────────────────────────────
    -- CAR FLY CONTROLLER
    -- ──────────────────────────────────────────────────────────────
    local CarFlyConfig = {
        Enabled          = false,
        Speed            = 120,
        Keybind          = Enum.KeyCode.H,
        AntiAimEnabled   = false,
        AntiAimMode      = "Jitter",
        AntiAimIntensity = 15,
        AntiAimSpeed     = 5,
        SpinEnabled      = false,
        SpinSpeed        = 20,
        SpinAxis         = "Yaw",
        SpinDirection    = 1,
        SpinJitter       = false,
        AutoLevel        = true,
        BoostMultiplier  = 2,
        VerticalControl  = true,
    }

    local function CreateCarFlyUI()
        local existing = Svc.CG:FindFirstChild("CarFlyController")
        if existing then existing:Destroy() end

        local ScreenGui = Instance.new("ScreenGui")
        ScreenGui.Name           = "CarFlyController"
        ScreenGui.ResetOnSpawn   = false
        ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
        ScreenGui.Parent         = Svc.CG

        local Main = Instance.new("Frame", ScreenGui)
        Main.Name              = "Main"
        Main.Size              = UDim2.new(0, 400, 0, 600)
        Main.Position          = UDim2.new(0.5, -200, 0.5, -300)
        Main.BackgroundColor3  = Color3.fromRGB(15, 15, 20)
        Main.BackgroundTransparency = 0.1
        Main.BorderSizePixel   = 0

        local Gradient = Instance.new("UIGradient", Main)
        Gradient.Color    = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(20, 20, 30)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(10, 10, 15)),
        })
        Gradient.Rotation = 135

        Instance.new("UICorner", Main).CornerRadius = UDim.new(0, 20)

        local Stroke = Instance.new("UIStroke", Main)
        Stroke.Color       = Color3.fromRGB(0, 255, 200)
        Stroke.Thickness   = 2
        Stroke.Transparency = 0.3

        local Shadow = Instance.new("ImageLabel", Main)
        Shadow.Size               = UDim2.new(1, 60, 1, 60)
        Shadow.Position           = UDim2.new(0, -30, 0, -30)
        Shadow.BackgroundTransparency = 1
        Shadow.Image              = "rbxassetid://5554236806"
        Shadow.ImageColor3        = Color3.fromRGB(0, 0, 0)
        Shadow.ImageTransparency  = 0.6
        Shadow.ScaleType          = Enum.ScaleType.Slice
        Shadow.SliceCenter        = Rect.new(30, 30, 270, 270)
        Shadow.ZIndex             = -1

        local Header = Instance.new("Frame", Main)
        Header.Size                  = UDim2.new(1, 0, 0, 60)
        Header.BackgroundColor3      = Color3.fromRGB(25, 25, 35)
        Header.BackgroundTransparency = 0.3
        Header.BorderSizePixel       = 0
        Instance.new("UICorner", Header).CornerRadius = UDim.new(0, 20)

        local HeaderFix = Instance.new("Frame", Header)
        HeaderFix.Size                  = UDim2.new(1, 0, 0.5, 0)
        HeaderFix.Position              = UDim2.new(0, 0, 0.5, 0)
        HeaderFix.BackgroundColor3      = Header.BackgroundColor3
        HeaderFix.BackgroundTransparency = Header.BackgroundTransparency
        HeaderFix.BorderSizePixel       = 0

        local Title = Instance.new("TextLabel", Header)
        Title.Size             = UDim2.new(0.6, 0, 1, 0)
        Title.Position         = UDim2.new(0.2, 0, 0, 0)
        Title.BackgroundTransparency = 1
        Title.Text             = "CAR CONTROLLER v2.0"
        Title.TextColor3       = Color3.fromRGB(0, 255, 200)
        Title.TextSize         = 22
        Title.Font             = Enum.Font.GothamBlack

        local StatusDot = Instance.new("Frame", Header)
        StatusDot.Size             = UDim2.new(0, 12, 0, 12)
        StatusDot.Position         = UDim2.new(0, 25, 0.5, -6)
        StatusDot.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
        StatusDot.BorderSizePixel  = 0
        Instance.new("UICorner", StatusDot).CornerRadius = UDim.new(1, 0)

        local CloseBtn = Instance.new("TextButton", Header)
        CloseBtn.Size              = UDim2.new(0, 35, 0, 35)
        CloseBtn.Position          = UDim2.new(1, -45, 0.5, -17.5)
        CloseBtn.BackgroundColor3  = Color3.fromRGB(255, 50, 50)
        CloseBtn.BackgroundTransparency = 0.8
        CloseBtn.Text              = "×"
        CloseBtn.TextColor3        = Color3.fromRGB(255, 255, 255)
        CloseBtn.TextSize          = 26
        CloseBtn.Font              = Enum.Font.GothamBold
        Instance.new("UICorner", CloseBtn).CornerRadius = UDim.new(0, 10)

        local Content = Instance.new("ScrollingFrame", Main)
        Content.Size              = UDim2.new(1, -20, 1, -80)
        Content.Position          = UDim2.new(0, 10, 0, 70)
        Content.BackgroundTransparency = 1
        Content.BorderSizePixel   = 0
        Content.ScrollBarThickness = 4
        Content.ScrollBarImageColor3 = Color3.fromRGB(0, 255, 200)
        Content.CanvasSize        = UDim2.new(0, 0, 0, 850)

        -- Toggle Section
        local ToggleSection = Instance.new("Frame", Content)
        ToggleSection.Size = UDim2.new(1, 0, 0, 100)
        ToggleSection.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
        ToggleSection.BackgroundTransparency = 0.5
        ToggleSection.BorderSizePixel = 0
        Instance.new("UICorner", ToggleSection).CornerRadius = UDim.new(0, 15)

        local PowerBtn = Instance.new("TextButton", ToggleSection)
        PowerBtn.Name              = "PowerBtn"
        PowerBtn.Size              = UDim2.new(0, 80, 0, 80)
        PowerBtn.Position          = UDim2.new(0.5, -40, 0.5, -40)
        PowerBtn.BackgroundColor3  = Color3.fromRGB(40, 40, 50)
        PowerBtn.Text              = "OFF"
        PowerBtn.TextColor3        = Color3.fromRGB(150, 150, 150)
        PowerBtn.TextSize          = 18
        PowerBtn.Font              = Enum.Font.GothamBlack
        Instance.new("UICorner", PowerBtn).CornerRadius = UDim.new(1, 0)

        local PowerStroke = Instance.new("UIStroke", PowerBtn)
        PowerStroke.Color     = Color3.fromRGB(100, 100, 100)
        PowerStroke.Thickness = 4

        local Glow = Instance.new("ImageLabel", PowerBtn)
        Glow.Size             = UDim2.new(1.4, 0, 1.4, 0)
        Glow.Position         = UDim2.new(-0.2, 0, -0.2, 0)
        Glow.BackgroundTransparency = 1
        Glow.Image            = "rbxassetid://10822646377"
        Glow.ImageColor3      = Color3.fromRGB(0, 255, 100)
        Glow.ImageTransparency = 1

        local StatusText = Instance.new("TextLabel", ToggleSection)
        StatusText.Name       = "StatusText"
        StatusText.Size       = UDim2.new(0, 100, 0, 20)
        StatusText.Position   = UDim2.new(0.5, -50, 1, -25)
        StatusText.BackgroundTransparency = 1
        StatusText.Text       = "DISCONNECTED"
        StatusText.TextColor3 = Color3.fromRGB(150, 150, 150)
        StatusText.TextSize   = 11
        StatusText.Font       = Enum.Font.GothamBold

        -- Speed Section
        local SpeedSection = Instance.new("Frame", Content)
        SpeedSection.Size     = UDim2.new(1, 0, 0, 100)
        SpeedSection.Position = UDim2.new(0, 0, 0, 110)
        SpeedSection.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
        SpeedSection.BackgroundTransparency = 0.5
        SpeedSection.BorderSizePixel = 0
        Instance.new("UICorner", SpeedSection).CornerRadius = UDim.new(0, 15)

        local SpeedLabel = Instance.new("TextLabel", SpeedSection)
        SpeedLabel.Size = UDim2.new(0, 100, 0, 25); SpeedLabel.Position = UDim2.new(0, 15, 0, 10)
        SpeedLabel.BackgroundTransparency = 1; SpeedLabel.Text = "FLIGHT SPEED"
        SpeedLabel.TextColor3 = Color3.fromRGB(200, 200, 200); SpeedLabel.TextSize = 12
        SpeedLabel.Font = Enum.Font.GothamBold; SpeedLabel.TextXAlignment = Enum.TextXAlignment.Left

        local SpeedValue = Instance.new("TextLabel", SpeedSection)
        SpeedValue.Name = "SpeedValue"; SpeedValue.Size = UDim2.new(0, 60, 0, 25)
        SpeedValue.Position = UDim2.new(1, -75, 0, 10); SpeedValue.BackgroundTransparency = 1
        SpeedValue.Text = "120"; SpeedValue.TextColor3 = Color3.fromRGB(0, 255, 200)
        SpeedValue.TextSize = 14; SpeedValue.Font = Enum.Font.GothamBlack
        SpeedValue.TextXAlignment = Enum.TextXAlignment.Right

        local SpeedSliderBg = Instance.new("Frame", SpeedSection)
        SpeedSliderBg.Size = UDim2.new(1, -30, 0, 8); SpeedSliderBg.Position = UDim2.new(0, 15, 0, 45)
        SpeedSliderBg.BackgroundColor3 = Color3.fromRGB(20, 20, 30); SpeedSliderBg.BorderSizePixel = 0
        Instance.new("UICorner", SpeedSliderBg).CornerRadius = UDim.new(1, 0)

        local SpeedFill = Instance.new("Frame", SpeedSliderBg)
        SpeedFill.Name = "SpeedFill"; SpeedFill.Size = UDim2.new(0.4, 0, 1, 0)
        SpeedFill.BackgroundColor3 = Color3.fromRGB(0, 255, 200); SpeedFill.BorderSizePixel = 0
        Instance.new("UICorner", SpeedFill).CornerRadius = UDim.new(1, 0)

        local SpeedKnob = Instance.new("TextButton", SpeedSliderBg)
        SpeedKnob.Name = "SpeedKnob"; SpeedKnob.Size = UDim2.new(0, 20, 0, 20)
        SpeedKnob.Position = UDim2.new(0.4, -10, 0.5, -10)
        SpeedKnob.BackgroundColor3 = Color3.fromRGB(255, 255, 255); SpeedKnob.Text = ""
        Instance.new("UICorner", SpeedKnob).CornerRadius = UDim.new(1, 0)

        local BoostBtn = Instance.new("TextButton", SpeedSection)
        BoostBtn.Size = UDim2.new(0, 80, 0, 30); BoostBtn.Position = UDim2.new(0.5, -40, 0, 65)
        BoostBtn.BackgroundColor3 = Color3.fromRGB(255, 150, 0); BoostBtn.BackgroundTransparency = 0.7
        BoostBtn.Text = "BOOST"; BoostBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        BoostBtn.TextSize = 12; BoostBtn.Font = Enum.Font.GothamBold
        Instance.new("UICorner", BoostBtn).CornerRadius = UDim.new(0, 8)

        -- Anti-Aim Section
        local AntiAimSection = Instance.new("Frame", Content)
        AntiAimSection.Size = UDim2.new(1, 0, 0, 220); AntiAimSection.Position = UDim2.new(0, 0, 0, 220)
        AntiAimSection.BackgroundColor3 = Color3.fromRGB(30, 30, 40); AntiAimSection.BackgroundTransparency = 0.5
        AntiAimSection.BorderSizePixel = 0
        Instance.new("UICorner", AntiAimSection).CornerRadius = UDim.new(0, 15)

        local AntiAimTitle = Instance.new("TextLabel", AntiAimSection)
        AntiAimTitle.Size = UDim2.new(1, -20, 0, 25); AntiAimTitle.Position = UDim2.new(0, 10, 0, 10)
        AntiAimTitle.BackgroundTransparency = 1; AntiAimTitle.Text = "ANTI-AIM // PITCH"
        AntiAimTitle.TextColor3 = Color3.fromRGB(255, 50, 150); AntiAimTitle.TextSize = 13
        AntiAimTitle.Font = Enum.Font.GothamBlack; AntiAimTitle.TextXAlignment = Enum.TextXAlignment.Left

        local AntiAimToggle = Instance.new("TextButton", AntiAimSection)
        AntiAimToggle.Name = "AntiAimToggle"; AntiAimToggle.Size = UDim2.new(0, 50, 0, 26)
        AntiAimToggle.Position = UDim2.new(1, -60, 0, 10); AntiAimToggle.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
        AntiAimToggle.Text = "OFF"; AntiAimToggle.TextColor3 = Color3.fromRGB(150, 150, 150)
        AntiAimToggle.TextSize = 11; AntiAimToggle.Font = Enum.Font.GothamBold
        Instance.new("UICorner", AntiAimToggle).CornerRadius = UDim.new(0, 13)

        local ModeBtn = Instance.new("TextButton", AntiAimSection)
        ModeBtn.Name = "ModeBtn"; ModeBtn.Size = UDim2.new(0, 100, 0, 28)
        ModeBtn.Position = UDim2.new(0, 15, 0, 65); ModeBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
        ModeBtn.Text = "Jitter"; ModeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        ModeBtn.TextSize = 12; ModeBtn.Font = Enum.Font.GothamBold
        Instance.new("UICorner", ModeBtn).CornerRadius = UDim.new(0, 8)

        local IntensitySlider = Instance.new("Frame", AntiAimSection)
        IntensitySlider.Size = UDim2.new(1, -30, 0, 6); IntensitySlider.Position = UDim2.new(0, 15, 0, 125)
        IntensitySlider.BackgroundColor3 = Color3.fromRGB(20, 20, 30); IntensitySlider.BorderSizePixel = 0

        local IntensityValue = Instance.new("TextLabel", AntiAimSection)
        IntensityValue.Name = "IntensityValue"; IntensityValue.Size = UDim2.new(0, 40, 0, 20)
        IntensityValue.Position = UDim2.new(1, -55, 0, 100); IntensityValue.BackgroundTransparency = 1
        IntensityValue.Text = "15°"; IntensityValue.TextColor3 = Color3.fromRGB(255, 50, 150)
        IntensityValue.TextSize = 12; IntensityValue.Font = Enum.Font.GothamBlack
        IntensityValue.TextXAlignment = Enum.TextXAlignment.Right

        local IntensityFill = Instance.new("Frame", IntensitySlider)
        IntensityFill.Size = UDim2.new(0.15, 0, 1, 0); IntensityFill.BackgroundColor3 = Color3.fromRGB(255, 50, 150)
        IntensityFill.BorderSizePixel = 0

        local IntensityKnob = Instance.new("TextButton", IntensitySlider)
        IntensityKnob.Size = UDim2.new(0, 16, 0, 16); IntensityKnob.Position = UDim2.new(0.15, -8, 0.5, -8)
        IntensityKnob.BackgroundColor3 = Color3.fromRGB(255, 255, 255); IntensityKnob.Text = ""

        local AASpeedSlider = Instance.new("Frame", AntiAimSection)
        AASpeedSlider.Size = UDim2.new(1, -30, 0, 6); AASpeedSlider.Position = UDim2.new(0, 15, 0, 170)
        AASpeedSlider.BackgroundColor3 = Color3.fromRGB(20, 20, 30); AASpeedSlider.BorderSizePixel = 0

        local AASpeedValue = Instance.new("TextLabel", AntiAimSection)
        AASpeedValue.Size = UDim2.new(0, 40, 0, 20); AASpeedValue.Position = UDim2.new(1, -55, 0, 145)
        AASpeedValue.BackgroundTransparency = 1; AASpeedValue.Text = "5"
        AASpeedValue.TextColor3 = Color3.fromRGB(255, 50, 150); AASpeedValue.TextSize = 12
        AASpeedValue.Font = Enum.Font.GothamBlack; AASpeedValue.TextXAlignment = Enum.TextXAlignment.Right

        local AASpeedFill = Instance.new("Frame", AASpeedSlider)
        AASpeedFill.Size = UDim2.new(0.25, 0, 1, 0); AASpeedFill.BackgroundColor3 = Color3.fromRGB(255, 50, 150)
        AASpeedFill.BorderSizePixel = 0

        local AASpeedKnob = Instance.new("TextButton", AASpeedSlider)
        AASpeedKnob.Size = UDim2.new(0, 16, 0, 16); AASpeedKnob.Position = UDim2.new(0.25, -8, 0.5, -8)
        AASpeedKnob.BackgroundColor3 = Color3.fromRGB(255, 255, 255); AASpeedKnob.Text = ""

        local CarIcon = Instance.new("TextLabel", AntiAimSection)
        CarIcon.Size = UDim2.new(0, 60, 0, 40); CarIcon.Position = UDim2.new(1, -75, 0, 55)
        CarIcon.BackgroundTransparency = 1; CarIcon.Text = "⬆"
        CarIcon.TextColor3 = Color3.fromRGB(255, 50, 150); CarIcon.TextSize = 24
        CarIcon.Font = Enum.Font.GothamBlack

        -- Spin Section
        local SpinSection = Instance.new("Frame", Content)
        SpinSection.Size = UDim2.new(1, 0, 0, 280); SpinSection.Position = UDim2.new(0, 0, 0, 450)
        SpinSection.BackgroundColor3 = Color3.fromRGB(30, 30, 40); SpinSection.BackgroundTransparency = 0.5
        SpinSection.BorderSizePixel = 0
        Instance.new("UICorner", SpinSection).CornerRadius = UDim.new(0, 15)

        local SpinTitle = Instance.new("TextLabel", SpinSection)
        SpinTitle.Size = UDim2.new(1, -20, 0, 25); SpinTitle.Position = UDim2.new(0, 10, 0, 10)
        SpinTitle.BackgroundTransparency = 1; SpinTitle.Text = "SPINBOT // ROTATION"
        SpinTitle.TextColor3 = Color3.fromRGB(100, 200, 255); SpinTitle.TextSize = 13
        SpinTitle.Font = Enum.Font.GothamBlack; SpinTitle.TextXAlignment = Enum.TextXAlignment.Left

        local SpinToggle = Instance.new("TextButton", SpinSection)
        SpinToggle.Name = "SpinToggle"; SpinToggle.Size = UDim2.new(0, 50, 0, 26)
        SpinToggle.Position = UDim2.new(1, -60, 0, 10); SpinToggle.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
        SpinToggle.Text = "OFF"; SpinToggle.TextColor3 = Color3.fromRGB(150, 150, 150)
        SpinToggle.TextSize = 11; SpinToggle.Font = Enum.Font.GothamBold
        Instance.new("UICorner", SpinToggle).CornerRadius = UDim.new(0, 13)

        local RPMDisplay = Instance.new("TextLabel", SpinSection)
        RPMDisplay.Size = UDim2.new(0, 100, 0, 30); RPMDisplay.Position = UDim2.new(0.5, -50, 0, 40)
        RPMDisplay.BackgroundColor3 = Color3.fromRGB(10, 10, 20); RPMDisplay.BackgroundTransparency = 0.5
        RPMDisplay.Text = "0 RPM"; RPMDisplay.TextColor3 = Color3.fromRGB(100, 200, 255)
        RPMDisplay.TextSize = 18; RPMDisplay.Font = Enum.Font.GothamBlack
        Instance.new("UICorner", RPMDisplay).CornerRadius = UDim.new(0, 8)

        local AxisBtn = Instance.new("TextButton", SpinSection)
        AxisBtn.Name = "AxisBtn"; AxisBtn.Size = UDim2.new(0, 100, 0, 28)
        AxisBtn.Position = UDim2.new(0, 15, 0, 100); AxisBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
        AxisBtn.Text = "Yaw"; AxisBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        AxisBtn.TextSize = 12; AxisBtn.Font = Enum.Font.GothamBold
        Instance.new("UICorner", AxisBtn).CornerRadius = UDim.new(0, 8)

        local DirBtn = Instance.new("TextButton", SpinSection)
        DirBtn.Name = "DirBtn"; DirBtn.Size = UDim2.new(0, 80, 0, 28)
        DirBtn.Position = UDim2.new(0, 125, 0, 100); DirBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
        DirBtn.Text = "CW"; DirBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        DirBtn.TextSize = 12; DirBtn.Font = Enum.Font.GothamBold
        Instance.new("UICorner", DirBtn).CornerRadius = UDim.new(0, 8)

        local SpinSpeedSlider = Instance.new("Frame", SpinSection)
        SpinSpeedSlider.Size = UDim2.new(1, -30, 0, 8); SpinSpeedSlider.Position = UDim2.new(0, 15, 0, 165)
        SpinSpeedSlider.BackgroundColor3 = Color3.fromRGB(20, 20, 30); SpinSpeedSlider.BorderSizePixel = 0

        local SpinSpeedValue = Instance.new("TextLabel", SpinSection)
        SpinSpeedValue.Size = UDim2.new(0, 50, 0, 20); SpinSpeedValue.Position = UDim2.new(1, -65, 0, 140)
        SpinSpeedValue.BackgroundTransparency = 1; SpinSpeedValue.Text = "20 RPS"
        SpinSpeedValue.TextColor3 = Color3.fromRGB(100, 200, 255); SpinSpeedValue.TextSize = 12
        SpinSpeedValue.Font = Enum.Font.GothamBlack; SpinSpeedValue.TextXAlignment = Enum.TextXAlignment.Right

        local SpinSpeedFill = Instance.new("Frame", SpinSpeedSlider)
        SpinSpeedFill.Size = UDim2.new(0.2, 0, 1, 0); SpinSpeedFill.BackgroundColor3 = Color3.fromRGB(100, 200, 255)
        SpinSpeedFill.BorderSizePixel = 0

        local SpinSpeedKnob = Instance.new("TextButton", SpinSpeedSlider)
        SpinSpeedKnob.Size = UDim2.new(0, 18, 0, 18); SpinSpeedKnob.Position = UDim2.new(0.2, -9, 0.5, -9)
        SpinSpeedKnob.BackgroundColor3 = Color3.fromRGB(255, 255, 255); SpinSpeedKnob.Text = ""

        local JitterBtn = Instance.new("TextButton", SpinSection)
        JitterBtn.Name = "JitterBtn"; JitterBtn.Size = UDim2.new(0, 120, 0, 30)
        JitterBtn.Position = UDim2.new(0.5, -60, 0, 195); JitterBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
        JitterBtn.Text = "JITTER: OFF"; JitterBtn.TextColor3 = Color3.fromRGB(150, 150, 150)
        JitterBtn.TextSize = 12; JitterBtn.Font = Enum.Font.GothamBold
        Instance.new("UICorner", JitterBtn).CornerRadius = UDim.new(0, 8)

        local SpinnerFrame = Instance.new("Frame", SpinSection)
        SpinnerFrame.Size = UDim2.new(0, 80, 0, 80); SpinnerFrame.Position = UDim2.new(1, -95, 0, 80)
        SpinnerFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 30); SpinnerFrame.BorderSizePixel = 0
        Instance.new("UICorner", SpinnerFrame).CornerRadius = UDim.new(1, 0)

        local SpinnerIcon = Instance.new("TextLabel", SpinnerFrame)
        SpinnerIcon.Size = UDim2.new(1, 0, 1, 0); SpinnerIcon.BackgroundTransparency = 1
        SpinnerIcon.Text = "⟳"; SpinnerIcon.TextColor3 = Color3.fromRGB(100, 200, 255)
        SpinnerIcon.TextSize = 40; SpinnerIcon.Font = Enum.Font.GothamBlack

        local SpinnerGlow = Instance.new("ImageLabel", SpinnerFrame)
        SpinnerGlow.Size = UDim2.new(1.5, 0, 1.5, 0); SpinnerGlow.Position = UDim2.new(-0.25, 0, -0.25, 0)
        SpinnerGlow.BackgroundTransparency = 1; SpinnerGlow.Image = "rbxassetid://10822646377"
        SpinnerGlow.ImageColor3 = Color3.fromRGB(100, 200, 255); SpinnerGlow.ImageTransparency = 1

        -- Keybind info
        local KeybindSection = Instance.new("Frame", Content)
        KeybindSection.Size = UDim2.new(1, 0, 0, 100); KeybindSection.Position = UDim2.new(0, 0, 0, 740)
        KeybindSection.BackgroundColor3 = Color3.fromRGB(30, 30, 40); KeybindSection.BackgroundTransparency = 0.5
        KeybindSection.BorderSizePixel = 0
        Instance.new("UICorner", KeybindSection).CornerRadius = UDim.new(0, 15)

        local KeybindText = Instance.new("TextLabel", KeybindSection)
        KeybindText.Size = UDim2.new(1, -20, 1, -15); KeybindText.Position = UDim2.new(0, 10, 0, 10)
        KeybindText.BackgroundTransparency = 1
        KeybindText.Text = "[H] Toggle Flight\n[Space/Ctrl] Up/Down  |  [Shift] Boost\nAnti-Aim: Pitch jitter  |  Spinbot: Rapid rotation"
        KeybindText.TextColor3 = Color3.fromRGB(150, 150, 150); KeybindText.TextSize = 11
        KeybindText.Font = Enum.Font.Gotham; KeybindText.TextXAlignment = Enum.TextXAlignment.Left

        -- ── Logic ────────────────────────────────────────────────
        local function UpdatePowerVisual()
            if CarFlyConfig.Enabled then
                PowerBtn.Text = "ON"; PowerBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 100)
                PowerStroke.Color = Color3.fromRGB(0, 255, 150)
                StatusText.Text = "CONNECTED"; StatusText.TextColor3 = Color3.fromRGB(0, 255, 150)
                StatusDot.BackgroundColor3 = Color3.fromRGB(0, 255, 100)
                task.spawn(function()
                    while CarFlyConfig.Enabled do
                        for i = 30, 80, 5 do
                            if not CarFlyConfig.Enabled then break end
                            Glow.ImageTransparency = i / 100; task.wait(0.1)
                        end
                        for i = 80, 30, -5 do
                            if not CarFlyConfig.Enabled then break end
                            Glow.ImageTransparency = i / 100; task.wait(0.1)
                        end
                    end
                end)
            else
                PowerBtn.Text = "OFF"; PowerBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
                PowerStroke.Color = Color3.fromRGB(100, 100, 100)
                StatusText.Text = "DISCONNECTED"; StatusText.TextColor3 = Color3.fromRGB(150, 150, 150)
                StatusDot.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
                Glow.ImageTransparency = 1
            end
        end

        -- Dragging
        local dragging, dragStart, startPos, dragInput = false, nil, nil, nil
        Header.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 then
                dragging = true; dragStart = input.Position; startPos = Main.Position
            end
        end)
        Header.InputChanged:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseMovement then dragInput = input end
        end)
        Svc.UIS.InputChanged:Connect(function(input)
            if input == dragInput and dragging then
                local d = input.Position - dragStart
                Main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
            end
        end)
        Svc.UIS.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
        end)

        CloseBtn.MouseButton1Click:Connect(function() ScreenGui:Destroy() end)
        PowerBtn.MouseButton1Click:Connect(function()
            CarFlyConfig.Enabled = not CarFlyConfig.Enabled; UpdatePowerVisual()
        end)
        AntiAimToggle.MouseButton1Click:Connect(function()
            CarFlyConfig.AntiAimEnabled = not CarFlyConfig.AntiAimEnabled
            AntiAimToggle.Text = CarFlyConfig.AntiAimEnabled and "ON" or "OFF"
            AntiAimToggle.BackgroundColor3 = CarFlyConfig.AntiAimEnabled and Color3.fromRGB(255, 50, 150) or Color3.fromRGB(60, 60, 70)
        end)
        local modes = {"Jitter","Sine","Random"}; local modeIdx = 1
        ModeBtn.MouseButton1Click:Connect(function()
            modeIdx = modeIdx % #modes + 1; CarFlyConfig.AntiAimMode = modes[modeIdx]; ModeBtn.Text = modes[modeIdx]
        end)
        SpinToggle.MouseButton1Click:Connect(function()
            CarFlyConfig.SpinEnabled = not CarFlyConfig.SpinEnabled
            SpinToggle.Text = CarFlyConfig.SpinEnabled and "ON" or "OFF"
            SpinToggle.BackgroundColor3 = CarFlyConfig.SpinEnabled and Color3.fromRGB(100, 200, 255) or Color3.fromRGB(60, 60, 70)
            if CarFlyConfig.SpinEnabled then
                task.spawn(function()
                    while CarFlyConfig.SpinEnabled do
                        SpinnerIcon.Rotation = (SpinnerIcon.Rotation + 15 * CarFlyConfig.SpinDirection) % 360
                        SpinnerGlow.ImageTransparency = 0.5 + math.sin(tick() * 10) * 0.3
                        task.wait(0.03)
                    end
                    SpinnerGlow.ImageTransparency = 1
                end)
            end
        end)
        local axes = {"Yaw","Pitch","Roll","All"}; local axisIdx = 1
        AxisBtn.MouseButton1Click:Connect(function()
            axisIdx = axisIdx % #axes + 1; CarFlyConfig.SpinAxis = axes[axisIdx]; AxisBtn.Text = axes[axisIdx]
        end)
        DirBtn.MouseButton1Click:Connect(function()
            CarFlyConfig.SpinDirection = -CarFlyConfig.SpinDirection
            DirBtn.Text = CarFlyConfig.SpinDirection == 1 and "CW" or "CCW"
        end)
        JitterBtn.MouseButton1Click:Connect(function()
            CarFlyConfig.SpinJitter = not CarFlyConfig.SpinJitter
            JitterBtn.Text = CarFlyConfig.SpinJitter and "JITTER: ON" or "JITTER: OFF"
            JitterBtn.TextColor3 = CarFlyConfig.SpinJitter and Color3.fromRGB(100, 200, 255) or Color3.fromRGB(150, 150, 150)
        end)

        local function SetupSlider(knob, fill, valueLabel, min, max, suffix, configKey, isDecimal)
            local sliding = false
            local function Update(input)
                local pos = math.clamp((input.Position.X - knob.Parent.AbsolutePosition.X) / knob.Parent.AbsoluteSize.X, 0, 1)
                fill.Size = UDim2.new(pos, 0, 1, 0)
                knob.Position = UDim2.new(pos, -9, 0.5, -9)
                local value = min + (max - min) * pos
                value = isDecimal and math.floor(value * 10) / 10 or math.floor(value)
                CarFlyConfig[configKey] = value
                valueLabel.Text = value .. suffix
            end
            knob.InputBegan:Connect(function(i) if i.UserInputType == Enum.UserInputType.MouseButton1 then sliding = true end end)
            knob.Parent.InputBegan:Connect(function(i) if i.UserInputType == Enum.UserInputType.MouseButton1 then sliding = true; Update(i) end end)
            Svc.UIS.InputChanged:Connect(function(i) if sliding and i.UserInputType == Enum.UserInputType.MouseMovement then Update(i) end end)
            Svc.UIS.InputEnded:Connect(function(i) if i.UserInputType == Enum.UserInputType.MouseButton1 then sliding = false end end)
        end

        SetupSlider(SpeedKnob,      SpeedFill,      SpeedValue,      0,   500, "",     "Speed")
        SetupSlider(IntensityKnob,  IntensityFill,  IntensityValue,  0,   45,  "°",    "AntiAimIntensity")
        SetupSlider(AASpeedKnob,    AASpeedFill,    AASpeedValue,    0.1, 20,  "",     "AntiAimSpeed",  true)
        SetupSlider(SpinSpeedKnob,  SpinSpeedFill,  SpinSpeedValue,  1,   100, " RPS", "SpinSpeed")

        task.spawn(function()
            while task.wait(0.05) do
                if CarFlyConfig.AntiAimEnabled then
                    local t = tick() * CarFlyConfig.AntiAimSpeed
                    local pitch = 0
                    if CarFlyConfig.AntiAimMode == "Sine" then
                        pitch = math.sin(t) * CarFlyConfig.AntiAimIntensity
                    elseif CarFlyConfig.AntiAimMode == "Jitter" then
                        pitch = math.random(-CarFlyConfig.AntiAimIntensity, CarFlyConfig.AntiAimIntensity)
                    elseif CarFlyConfig.AntiAimMode == "Random" then
                        pitch = (math.noise(t * 0.5) * 2 - 1) * CarFlyConfig.AntiAimIntensity
                    end
                    if pitch > 5 then CarIcon.Text = "⬆"; CarIcon.TextColor3 = Color3.fromRGB(0, 255, 100)
                    elseif pitch < -5 then CarIcon.Text = "⬇"; CarIcon.TextColor3 = Color3.fromRGB(255, 50, 50)
                    else CarIcon.Text = "➡"; CarIcon.TextColor3 = Color3.fromRGB(255, 50, 150) end
                else
                    CarIcon.Text = "➡"; CarIcon.TextColor3 = Color3.fromRGB(100, 100, 100)
                end
            end
        end)

        task.spawn(function()
            while task.wait(0.1) do
                if CarFlyConfig.SpinEnabled then
                    local rpm = CarFlyConfig.SpinSpeed * 60 * (CarFlyConfig.SpinJitter and math.random(80, 120) / 100 or 1)
                    RPMDisplay.Text = math.floor(rpm) .. " RPM"
                else
                    RPMDisplay.Text = "0 RPM"
                end
            end
        end)

        return ScreenGui
    end

    local _carFlyUI      = nil
    local _cfAntiAimOff  = 0
    local _cfSpinAngle   = 0

    local function GetCurrentCar()
        for _, obj in pairs(Svc.WS:GetChildren()) do
            local chassis = obj:FindFirstChild("Chassis")
            local config  = obj:FindFirstChild("Configuration")
            if chassis and chassis:IsA("BasePart") and config then
                local dv = config:FindFirstChild("Driver")
                if dv and dv:IsA("ObjectValue") and dv.Value == LocalPlayer then
                    return chassis
                end
            end
        end
        return nil
    end

    _carFlyUI = CreateCarFlyUI()

    Svc.Run.Heartbeat:Connect(function(dt)
        if not CarFlyConfig.Enabled then
            _cfAntiAimOff = 0; _cfSpinAngle = 0; return
        end
        local chassis = GetCurrentCar()
        if not chassis then return end

        local cam   = Camera
        local cd    = cam.CFrame.LookVector
        local speed = CarFlyConfig.Speed
        if Svc.UIS:IsKeyDown(Enum.KeyCode.LeftShift) then speed = speed * CarFlyConfig.BoostMultiplier end

        local vert = 0
        if CarFlyConfig.VerticalControl then
            if Svc.UIS:IsKeyDown(Enum.KeyCode.Space)       then vert =  speed * 0.5 end
            if Svc.UIS:IsKeyDown(Enum.KeyCode.LeftControl) then vert = -speed * 0.5 end
        end

        chassis.AssemblyLinearVelocity = Vector3.new(cd.X * speed, cd.Y * speed + vert, cd.Z * speed)

        local hd = Vector3.new(cd.X, 0, cd.Z)
        local baseRot = (hd.Magnitude > 0.1) and CFrame.lookAt(Vector3.zero, hd) or CFrame.new()

        if CarFlyConfig.AntiAimEnabled then
            local t = tick() * CarFlyConfig.AntiAimSpeed
            if CarFlyConfig.AntiAimMode == "Sine" then
                _cfAntiAimOff = math.sin(t) * math.rad(CarFlyConfig.AntiAimIntensity)
            elseif CarFlyConfig.AntiAimMode == "Jitter" then
                if math.random() < 0.1 then
                    _cfAntiAimOff = (math.random() * 2 - 1) * math.rad(CarFlyConfig.AntiAimIntensity)
                end
            elseif CarFlyConfig.AntiAimMode == "Random" then
                _cfAntiAimOff = (math.noise(t * 0.5) * 2 - 1) * math.rad(CarFlyConfig.AntiAimIntensity)
            end
        else
            _cfAntiAimOff = 0
        end

        local pitchCF = CFrame.Angles(_cfAntiAimOff, 0, 0)

        if CarFlyConfig.SpinEnabled then
            local ss = CarFlyConfig.SpinSpeed * 360 * dt * CarFlyConfig.SpinDirection
            if CarFlyConfig.SpinJitter then ss = ss * (0.8 + math.random() * 0.4) end
            _cfSpinAngle = (_cfSpinAngle + ss) % 360
            local sa = math.rad(_cfSpinAngle)
            local spinCF = CFrame.new()
            if     CarFlyConfig.SpinAxis == "Yaw"   then spinCF = CFrame.Angles(0, sa, 0)
            elseif CarFlyConfig.SpinAxis == "Pitch" then spinCF = CFrame.Angles(sa, 0, 0)
            elseif CarFlyConfig.SpinAxis == "Roll"  then spinCF = CFrame.Angles(0, 0, sa)
            elseif CarFlyConfig.SpinAxis == "All"   then spinCF = CFrame.Angles(sa*0.7, sa, sa*0.5) end
            chassis.CFrame = CFrame.new(chassis.Position) * baseRot * pitchCF * spinCF
        else
            chassis.CFrame = CFrame.new(chassis.Position) * baseRot * pitchCF
        end
        chassis.AssemblyAngularVelocity = Vector3.zero
    end)

    Svc.UIS.InputBegan:Connect(function(input, gp)
        if gp then return end
        if input.KeyCode == CarFlyConfig.Keybind then
            CarFlyConfig.Enabled = not CarFlyConfig.Enabled
        end
    end)

    -- Obsidian tab button to reopen UI if closed
    local CFGroup = Tabs.Exploit:AddLeftGroupbox("Car Fly")
    CFGroup:AddToggle("CarFlyEnabled", {Text="Enable Car Fly", Default=false})
        :AddKeyPicker("CarFlyKey", {Default="H", NoUI=false, Text="Toggle Car Fly", Mode="Toggle"})
    Toggles.CarFlyEnabled:OnChanged(function()
        CarFlyConfig.Enabled = Toggles.CarFlyEnabled.Value
    end)
    Options.CarFlyKey:OnClick(function()
        Toggles.CarFlyEnabled:SetValue(not Toggles.CarFlyEnabled.Value)
    end)
    CFGroup:AddButton("Open Car Fly UI", function()
        local existing = Svc.CG:FindFirstChild("CarFlyController")
        if existing then existing:Destroy() end
        _carFlyUI = CreateCarFlyUI()
    end)


     do
    local dcs_ok, dcs = pcall(function()
        return require(Svc.RS.EmberSharedLibrary.GameShared.Services["DayCycleService.service"])
    end)
    local flags = {
        customWorldEnabled=false, customWorldBrightness=3,
        customWorldAmbient=Color3.fromRGB(70,70,70), customWorldOutdoorAmbient=Color3.fromRGB(70,70,70),
        customWorldColorShiftBottom=Color3.new(), customWorldColorShiftTop=Color3.new(),
        customWorldGlobalShadows=true, customWorldFogColor=Color3.fromRGB(192,192,192),
        customWorldFogEnd=10000, customWorldFogStart=0, customWorldClockTime=14.5,
        customAtmosphereEnabled=false, customAtmosphereDensity=0.28, customAtmosphereOffset=1,
        customAtmosphereColor=Color3.new(1,1,1), customAtmosphereDecay=Color3.new(),
        customAtmosphereGlare=1, customAtmosphereHaze=1,
    }
    if dcs_ok and dcs then
        local old_dcs; old_dcs = hookfunction(dcs.GetExpectedValues, newcclosure(function(self, p37)
            local va = old_dcs(self, p37)
            for i, pp in pairs(va) do
                if i:IsA("Lighting") and flags.customWorldEnabled then
                    pp.Brightness=flags.customWorldBrightness; pp.Ambient=flags.customWorldAmbient
                    pp.OutdoorAmbient=flags.customWorldOutdoorAmbient
                    pp.ColorShift_Bottom=flags.customWorldColorShiftBottom; pp.ColorShift_Top=flags.customWorldColorShiftTop
                    pp.GlobalShadows=flags.customWorldGlobalShadows; pp.FogColor=flags.customWorldFogColor
                    pp.FogEnd=flags.customWorldFogEnd; pp.FogStart=flags.customWorldFogStart; pp.ClockTime=flags.customWorldClockTime
                end
                if i:IsA("Atmosphere") and flags.customAtmosphereEnabled then
                    pp.Density=flags.customAtmosphereDensity; pp.Offset=flags.customAtmosphereOffset
                    pp.Color=flags.customAtmosphereColor; pp.Decay=flags.customAtmosphereDecay
                    pp.Glare=flags.customAtmosphereGlare; pp.Haze=flags.customAtmosphereHaze
                end
            end
            return va
        end))
    end

    local _treeconn, _foliageconn = nil, nil

    local function set_transparency(tree, transparency)
        if not tree then return end
        if tree:IsA("Model") then
            local leaves = tree:FindFirstChild("Leaves")
            if leaves and leaves.Transparency ~= transparency then leaves.Transparency = transparency end
        end
        if tree:IsA("BasePart") and tree.CanCollide == false and tree.Transparency ~= transparency then
            tree.Transparency = transparency
        end
    end

    local world = Tabs.Exploit:AddRightTabbox()
    local lighting_tab = world:AddTab("lighting")
    local world_tab = world:AddTab("world")

    lighting_tab:AddToggle('fullbright_toggle',{Text='Lighting Changer',Default=false,Callback=function(s) flags.customWorldEnabled=s end})
    lighting_tab:AddLabel("Ambient"):AddColorPicker('ambient_color',{Default=Color3.fromRGB(70,70,70),Title='Ambient',Callback=function(c) flags.customWorldAmbient=c end})
    lighting_tab:AddLabel("Outdoor Ambient"):AddColorPicker('outdoor_ambient_color',{Default=Color3.fromRGB(70,70,70),Title='Outdoor Ambient',Callback=function(c) flags.customWorldOutdoorAmbient=c end})
    lighting_tab:AddSlider('brightness_slider',{Text='Brightness',Default=3,Min=0,Max=10,Rounding=2,Callback=function(v) flags.customWorldBrightness=v end})
    lighting_tab:AddToggle('global_shadows_toggle',{Text='Global Shadows',Default=true,Callback=function(s) flags.customWorldGlobalShadows=s end})
    lighting_tab:AddLabel("Fog Color"):AddColorPicker('fog_color',{Default=Color3.fromRGB(192,192,192),Title='Fog Color',Callback=function(c) flags.customWorldFogColor=c end})
    lighting_tab:AddSlider('clock_time_slider',{Text='Clock Time',Default=14.5,Min=0,Max=24,Rounding=1,Callback=function(v) flags.customWorldClockTime=v end})
    lighting_tab:AddToggle('atmosphere_toggle',{Text='Atmosphere Changer',Default=false,Callback=function(s) flags.customAtmosphereEnabled=s end})
    lighting_tab:AddSlider('atmosphere_density',{Text='Density',Default=0.28,Min=0,Max=1,Rounding=2,Callback=function(v) flags.customAtmosphereDensity=v end})
    lighting_tab:AddLabel("Atmosphere Color"):AddColorPicker('atmosphere_color',{Default=Color3.new(1,1,1),Title='Color',Callback=function(c) flags.customAtmosphereColor=c end})
    lighting_tab:AddLabel("Atmosphere Decay"):AddColorPicker('atmosphere_decay',{Default=Color3.new(),Title='Decay',Callback=function(c) flags.customAtmosphereDecay=c end})

    world_tab:AddToggle('world_grass',{Text='Grass',Default=true,Callback=function(s) sethiddenproperty(Svc.WS.Terrain,"Decoration",s) end})
    world_tab:AddSlider("FoliageTransparency", {
        Text = "Foliage Transparency",
        Default = 0,
        Min = 0,
        Max = 1,
        Rounding = 1,
        Callback = function(value)
            if _treeconn    then _treeconn:Disconnect();    _treeconn    = nil end
            if _foliageconn then _foliageconn:Disconnect(); _foliageconn = nil end
            local wa = Svc.WS:FindFirstChild("world_assets")
            if not wa then return end
            local so = wa:FindFirstChild("StaticObjects")
            if not so then return end
            local trees    = so:FindFirstChild("Trees")
            local foliages = so:FindFirstChild("Foliage")
            local function applyAll(folder, transparency)
                if not folder then return end
                for _, obj in ipairs(folder:GetChildren()) do
                    set_transparency(obj, transparency)
                end
            end
            applyAll(trees,    value)
            applyAll(foliages, value)
            local function onChildAdded(child) set_transparency(child, value) end
            if trees    then _treeconn    = trees.ChildAdded:Connect(onChildAdded) end
            if foliages then _foliageconn = foliages.ChildAdded:Connect(onChildAdded) end
        end
    })
end

        do
            local MenuGroup = Tabs.Settings:AddLeftGroupbox("Menu")
            MenuGroup:AddToggle("KeybindMenuOpen",{Default=Library.KeybindFrame.Visible,Text="Open Keybind Menu",Callback=function(v) Library.KeybindFrame.Visible=v end})
            MenuGroup:AddToggle("ShowCustomCursor",{Text="Custom Cursor",Default=true,Callback=function(v) Library.ShowCustomCursor=v end})
            MenuGroup:AddDropdown("NotificationSide",{Values={"Left","Right"},Default="Right",Text="Notification Side",Callback=function(v) Library:SetNotifySide(v) end})
            MenuGroup:AddDivider()
            MenuGroup:AddLabel("Menu Keybind"):AddKeyPicker("MenuKeybind",{Default="RightShift",NoUI=true,Text="Menu keybind"})
            MenuGroup:AddButton("Unload",function() Library:Unload() end)
            Library.ToggleKeybind = Options.MenuKeybind
            ThemeManager:SetLibrary(Library); SaveManager:SetLibrary(Library)
            SaveManager:IgnoreThemeSettings(); SaveManager:SetIgnoreIndexes({"MenuKeybind"})
            ThemeManager:SetFolder("AkkoMenu"); SaveManager:SetFolder("AkkoMenu/configs")
            SaveManager:BuildConfigSection(Tabs.Settings); ThemeManager:ApplyToTab(Tabs.Settings)
            SaveManager:LoadAutoloadConfig()
        end

        Library:OnUnload(function()
            local SH = State.SpeedHack
            stopPlayerESP()
            if _transpConn_add then _transpConn_add:Disconnect() end
            if _transpConn_rem then _transpConn_rem:Disconnect() end
            pcall(function() statusText:Remove() end)
            pcall(function() GB.square:Remove() end)
            for _, t in ipairs(GB.lines) do pcall(function() t:Remove() end) end
            if SH.conn then SH.conn:Disconnect(); SH.conn = nil end
            State.Config.NoRecoilEnabled = false; rawset(_G, "NoRecoilHooked", nil)
            pcall(function() set_aimbot_enabled(false) end)
            pcall(function() if old_buffer_create then hookfunction(buffer.create, old_buffer_create) end end)
            pcall(function() if fovcircle then fovcircle:Remove() end end)
            pcall(function() if mouse_aim_connection then mouse_aim_connection:Disconnect() end end)
            pcall(function() StopAntiAim() end)
            pcall(function() StopYaw() end)
            pcall(function() ToggleInstantAim(false) end)
            pcall(function() StopAntiAim() end)
            pcall(function() StopYaw() end)
            pcall(function() ToggleInstantAim(false) end)
            pcall(function()
                CarFlyConfig.Enabled = false
                local ui = Svc.CG:FindFirstChild("CarFlyController")
                if ui then ui:Destroy() end
            end)
            Library.Unloaded = true
        end)

        if Toggles.ESPEnabled and Toggles.ESPEnabled.Value then startPlayerESP() end
    end

    setupUI()
end

__main()

-- Robscript Team - Defeat Ragdolls Auto Farm
-- Game: Defeat Ragdolls for Powers | placeId 136950293103590
-- Window: RightControl | API: getgenv().RobScriptDR

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local Workspace = workspace

local LocalPlayer = Players.LocalPlayer

-- Unload a previous copy first, so its loop cannot keep punching behind the new one.
do
	local existing = getgenv().RobScriptDR
	if type(existing) == "table" and type(existing.Destroy) == "function" then
		pcall(existing.Destroy)
	end
end

----- Library -----

local LIBRARY_URLS = {
	"https://raw.githubusercontent.com/bobloscript/Robly_Airflow/main/Source.luau",
	"https://raw.githubusercontent.com/PookiePepelsss/Airflow-UI/refs/heads/main/Source.luau",
}

local Airflow = nil
local fallbackLibrary = nil
for _, url in ipairs(LIBRARY_URLS) do
	local ok, library = pcall(function()
		return loadstring(game:HttpGet(url))()
	end)
	if ok and type(library) == "table" and type(library.CreateWindow) == "function" then
		if not fallbackLibrary then
			fallbackLibrary = library
		end
		if type(library.ThemePresets) == "table" then
			Airflow = library
			break
		end
	end
end

Airflow = Airflow or fallbackLibrary

if not Airflow then
	error("[RobScript DR] Airflow UI could not be loaded")
end

----- Theme -----

local THEME = {
	Background = Color3.fromHex("#0A0A0C"),
	Surface = Color3.fromHex("#141418"),
	Surface2 = Color3.fromHex("#1A1A20"),
	Surface3 = Color3.fromHex("#2A2A33"),
	Stroke = Color3.fromHex("#26262E"),
	StrokeHover = Color3.fromHex("#3C3C49"),
	Accent = Color3.fromHex("#FF3B7F"),
	AccentDark = Color3.fromHex("#1A0710"),
	Text = Color3.fromHex("#F5F5F7"),
	Muted = Color3.fromHex("#8D8D99"),
}

local ThemeValue = THEME
do
	local merged = {}
	for key, value in pairs(Airflow.Theme or {}) do
		merged[key] = value
	end
	for key, value in pairs(THEME) do
		merged[key] = value
	end
	if type(Airflow.AddTheme) == "function" and type(Airflow.SetTheme) == "function" then
		local ok = pcall(function()
			Airflow:AddTheme("Robscript", merged)
			Airflow:SetTheme("Robscript")
		end)
		if ok then
			ThemeValue = "Robscript"
		end
	elseif type(Airflow.ThemePresets) == "table" then
		Airflow.ThemePresets.Robscript = merged
		pcall(function()
			Airflow:SetDefaultTheme("Robscript")
		end)
	end
end

----- Flags -----

local flagCache = {}

local function getFlag(name)
	local cached = flagCache[name]
	if cached ~= nil then
		return cached ~= false and cached or nil
	end
	local container = Airflow.Flags
	local entry = type(container) == "table" and container[name] or nil
	if type(entry) == "table" then
		flagCache[name] = entry
		return entry
	end
	-- The loop starts before the window is built, so a missing flag must not be
	-- cached: it is simply not registered yet.
	return nil
end

local function flag(name, fallback)
	local entry = getFlag(name)
	if not entry then
		return fallback
	end
	local ok, value = pcall(function()
		return entry:Get()
	end)
	if ok and value ~= nil then
		return value
	end
	return fallback
end

local function setFlag(name, value)
	local entry = getFlag(name)
	if not entry then
		return false
	end
	return (pcall(function()
		entry:Set(value)
	end))
end

----- Remotes -----

local Remotes = ReplicatedStorage:FindFirstChild("Remotes")
local PunchRemote = Remotes and Remotes:FindFirstChild("M1")
local HitRemote = Remotes and Remotes:FindFirstChild("M1Hits")
local RequestRemote = Remotes and Remotes:FindFirstChild("Request")

local Ready = PunchRemote ~= nil and HitRemote ~= nil and RequestRemote ~= nil

----- State -----

local running = true

local state = {
	punches = 0,
	kills = 0,
	claims = 0,
	target = "-",
	targetHp = 0,
}

local containers = {}
local containerStamp = 0
local lastPunch = 0
local lastMove = 0
local lastClaim = 0
local settleUntil = 0
local vacuumUntil = 0
local currentTarget = nil
local marked = {}
local glueTarget = nil
local stickEnabled = true
local stickRange = 3.5
local attackId = 1000
local combo = 0

----- Helpers -----

local function getCharacter()
	local character = LocalPlayer.Character
	if not character or not character.Parent then
		return nil
	end
	return character
end

local function getRoot()
	local character = getCharacter()
	if not character then
		return nil
	end
	return character:FindFirstChild("HumanoidRootPart")
end

local function getHumanoid()
	local character = getCharacter()
	if not character then
		return nil
	end
	return character:FindFirstChildOfClass("Humanoid")
end

-- The NPCs live in a handful of folders: Ragdolls in the city, Villains, Police
-- and whatever the stage system spawns. Rebuild the list from the live
-- humanoids, so a newly spawned container is picked up without a name list.
local function refreshContainers()
	if os.clock() - containerStamp < 4 then
		return containers
	end
	containerStamp = os.clock()
	local seen = {}
	local list = {}
	for _, descendant in ipairs(Workspace:GetDescendants()) do
		if descendant:IsA("Humanoid") then
			local model = descendant.Parent
			local holder = model and model.Parent
			if holder and holder ~= Workspace and not seen[holder] then
				seen[holder] = true
				list[#list + 1] = holder
			end
		end
	end
	containers = list
	return list
end

-- Every body part carries its own health in a Limb_* attribute. A hit on a
-- broken limb barely scratches the NPC, so the swing aims at the part that
-- still has the most left.
local LIMBS = {
	{ "Head", "Limb_Head" },
	{ "UpperTorso", "Limb_Torso" },
	{ "Torso", "Limb_Torso" },
	{ "LeftUpperArm", "Limb_LeftArm" },
	{ "RightUpperArm", "Limb_RightArm" },
	{ "LeftUpperLeg", "Limb_LeftLeg" },
	{ "RightUpperLeg", "Limb_RightLeg" },
}

local function getHitPart(model)
	local bestPart, bestHealth = nil, -1
	for _, entry in ipairs(LIMBS) do
		local part = model:FindFirstChild(entry[1])
		if part then
			local health = model:GetAttribute(entry[2]) or 0
			if health > bestHealth then
				bestPart, bestHealth = part, health
			end
		end
	end
	if bestPart then
		return bestPart
	end
	return model:FindFirstChild("Head")
		or model:FindFirstChild("UpperTorso")
		or model:FindFirstChild("Torso")
		or model:FindFirstChild("HumanoidRootPart")
		or model.PrimaryPart
end

-- A farmable NPC is a model with its own humanoid and a Health attribute that is
-- still above zero. Citizens strapped into a hazard and knocked out bodies are
-- skipped, they cannot be damaged any more.
local function isFarmable(model)
	if not model:IsA("Model") then
		return false
	end
	if model == getCharacter() then
		return false
	end
	if Players:GetPlayerFromCharacter(model) then
		return false
	end
	if model:GetAttribute("Strapped") or model:GetAttribute("KO") then
		return false
	end
	local humanoid = model:FindFirstChildOfClass("Humanoid")
	if not humanoid or humanoid.Health <= 0 then
		return false
	end
	local health = model:GetAttribute("Health")
	if type(health) ~= "number" or health <= 0 then
		return false
	end
	return true
end

local function findNearest(radius, root)
	local bestModel, bestPart, bestDistance = nil, nil, nil
	for _, holder in ipairs(refreshContainers()) do
		if holder.Parent then
			for _, model in ipairs(holder:GetChildren()) do
				if isFarmable(model) then
					local part = getHitPart(model)
					if part then
						local distance = (part.Position - root.Position).Magnitude
						if distance <= radius and (not bestDistance or distance < bestDistance) then
							bestModel, bestPart, bestDistance = model, part, distance
						end
					end
				end
			end
		end
	end
	return bestModel, bestPart, bestDistance
end

-- Stay on the NPC that is already being punched. Switching to whoever happens
-- to be one stud closer would split the damage over the whole crowd.
local function pickTarget(root, radius)
	if currentTarget and currentTarget.Parent and isFarmable(currentTarget) then
		local part = getHitPart(currentTarget)
		if part then
			local distance = (part.Position - root.Position).Magnitude
			if distance <= radius then
				return currentTarget, part, distance
			end
		end
	end
	return findNearest(radius, root)
end

-- Stand a little in front of the NPC and look at it. This mirrors what a player
-- does by walking up to the target, only without the walk.
local function approach(root, part, range)
	local direction = root.Position - part.Position
	if direction.Magnitude < 0.05 then
		direction = Vector3.new(0, 0, 1)
	end
	local from = part.Position + direction.Unit * range + Vector3.new(0, 0.5, 0)
	local look = Vector3.new(part.Position.X, from.Y, part.Position.Z)
	root.CFrame = CFrame.lookAt(from, look)
end

-- The client sends a swing index, a swing id, then the parts it hit. The server
-- validates the hit part and applies the damage, so the pair has to be fired in
-- that order with the same id.
local function punch(model, part)
	combo = combo % 3 + 1
	attackId = attackId + 1
	PunchRemote:FireServer(combo, attackId)
	task.wait(0.03)
	HitRemote:FireServer(attackId, { { r = math.random(), p = part } })
	marked[model] = true
end

----- Drops -----

-- Both drop kinds are claimed by the same remote, only the action and the
-- number in the name differ: Token623 -> Claim 623, Drumstick235 -> ClaimDrumstick 235.
local DROP_KINDS = {
	{ Folder = "TokenCoins", Action = "Claim" },
	{ Folder = "Drumsticks", Action = "ClaimDrumstick" },
}

-- A claim is only counted once the object is actually gone, so a rejected
-- request cannot inflate the counter.
local pending = {}

local function reapClaims()
	for object, info in pairs(pending) do
		if object.Parent == nil then
			pending[object] = nil
			-- Counted only when the drop vanished while the character was close:
			-- that is the client pickup or one of our claims, not an expiry on
			-- the other side of the map.
			if info.distance <= 25 then
				state.claims = state.claims + 1
			end
		elseif os.clock() - info.at > 5 then
			pending[object] = nil
		end
	end
end

local function gatherDrops(root, maxDistance)
	local list = {}
	for _, kind in ipairs(DROP_KINDS) do
		local folder = Workspace:FindFirstChild(kind.Folder)
		if folder then
			for _, object in ipairs(folder:GetChildren()) do
				local id = object.Name:match("(%d+)$")
				local part = object.PrimaryPart or object:FindFirstChildWhichIsA("BasePart")
				if id and part then
					local distance = (part.Position - root.Position).Magnitude
					if distance <= maxDistance then
						list[#list + 1] = {
							object = object,
							part = part,
							id = tonumber(id),
							action = kind.Action,
							distance = distance,
						}
					end
				end
			end
		end
	end
	table.sort(list, function(a, b)
		return a.distance < b.distance
	end)
	return list
end

-- The drops are server owned, so moving them from the client does nothing:
-- the character has to come within claim range and ask for them. The client
-- itself grabs everything inside its own radius, so the job here is the pile
-- that is left a few studs outside it.
local function collectDrops(allowHop)
	local root = getRoot()
	if not root then
		return {}
	end

	local claimRadius = math.clamp(flag("ClaimRadius", 6), 3, 10)
	local searchRadius = math.clamp(flag("CollectRadius", 60), 20, 200)
	local drops = gatherDrops(root, searchRadius)
	if #drops == 0 then
		return drops
	end

	if os.clock() - lastClaim >= 0.15 then
		lastClaim = os.clock()
		for _, drop in ipairs(drops) do
			local entry = pending[drop.object]
			if entry then
				entry.distance = drop.distance
			else
				pending[drop.object] = { at = os.clock(), distance = drop.distance }
			end
			if drop.distance <= claimRadius then
				RequestRemote:FireServer(drop.action, drop.id)
			end
		end
	end

	-- Nothing in reach: step the few studs to the nearest drop. The claim comes
	-- on the next pass, after the step has settled.
	local nearest = drops[1]
	if allowHop and flag("TeleportToDrops", true) and nearest.distance > claimRadius
		and os.clock() - lastMove > 0.5 then
		approach(root, nearest.part, 3)
		lastMove = os.clock()
		settleUntil = os.clock() + 0.4
	end

	return drops
end

----- Farm loop -----

local function step()
	local root = getRoot()
	if not root then
		return
	end
	local humanoid = getHumanoid()
	if humanoid and humanoid.Health <= 0 then
		return
	end

	reapClaims()

	-- The Health attribute updates a moment after the hit, so a knockout is
	-- counted when the NPC is seen dead, not on the punch that did it.
	for model in pairs(marked) do
		if model.Parent == nil or (model:GetAttribute("Health") or 0) <= 0 then
			marked[model] = nil
			state.kills = state.kills + 1
			-- Clear the pile the knockout left behind before running to the next NPC.
			vacuumUntil = os.clock() + 3
		end
	end

	-- Every step is followed by a short settle: the server checks punches and
	-- claims against the position it already knows.
	if os.clock() < settleUntil then
		return
	end

	local farming = flag("AutoFarm", false)
	if not farming then
		glueTarget = nil
	end
	if flag("AutoCollect", true) then
		collectDrops((not farming) or os.clock() < vacuumUntil)
	end
	if os.clock() < vacuumUntil then
		return
	end
	if not farming then
		return
	end

	stickEnabled = flag("StickToTarget", true)
	stickRange = math.clamp(flag("StickDistance", 3.5), 2, 6)

	local radius = math.clamp(flag("FarmRadius", 200), 30, 500)
	local punchRange = math.clamp(flag("PunchRange", 8), 3, 12)
	local delay = math.clamp(flag("PunchDelay", 0.45), 0.3, 1)

	local target, part, distance = pickTarget(root, radius)
	if not target then
		currentTarget = nil
		glueTarget = nil
		state.target = "none"
		state.targetHp = 0
		return
	end

	currentTarget = target
	glueTarget = target
	state.target = target.Name
	state.targetHp = math.floor(target:GetAttribute("Health") or 0)

	-- A long jump is still a jump: the server has to see the new position
	-- before it accepts a swing, so it is done once and left to settle.
	-- Everything closer is closed by the glue loop below.
	if distance > 40 then
		if flag("TeleportToTarget", true) then
			approach(root, part, math.max(stickRange, 3))
			lastMove = os.clock()
			settleUntil = os.clock() + 0.4
		elseif humanoid then
			humanoid:MoveTo(part.Position)
		end
		return
	end

	if flag("StickToTarget", true) then
		-- The glue loop keeps the character next to the NPC, so the swing waits
		-- for it to arrive instead of teleporting on every punch.
		if distance > punchRange then
			return
		end
	elseif distance > punchRange then
		if flag("TeleportToTarget", true) then
			approach(root, part, math.max(stickRange, 3))
			lastMove = os.clock()
			task.wait(0.4)
		elseif humanoid then
			humanoid:MoveTo(part.Position)
			return
		end
	end

	local fresh = getHitPart(target)
	if not fresh then
		return
	end
	part = fresh

	if (part.Position - root.Position).Magnitude > punchRange + 2 then
		return
	end

	if os.clock() - lastPunch < delay then
		return
	end

	punch(target, part)
	lastPunch = os.clock()
	state.punches = state.punches + 1
end

----- Glue -----

-- Instead of stepping to the NPC before every punch, the character is held next
-- to it every frame: same result as walking into it, without the jump that the
-- server would have to accept first.
local glueConnection = RunService.RenderStepped:Connect(function(delta)
	if not running or not stickEnabled then
		return
	end
	local target = glueTarget
	if not target or not target.Parent then
		return
	end
	local part = getHitPart(target)
	local root = getRoot()
	if not part or not root then
		return
	end
	local offset = root.Position - part.Position
	if offset.Magnitude < 0.1 then
		offset = Vector3.new(0, 0, 1)
	end
	local range = stickRange
	local from = part.Position + offset.Unit * range + Vector3.new(0, 0.5, 0)
	local goal = CFrame.lookAt(from, Vector3.new(part.Position.X, from.Y, part.Position.Z))
	root.CFrame = root.CFrame:Lerp(goal, math.clamp(delta * 14, 0, 1))
	root.AssemblyLinearVelocity = Vector3.zero
end)

task.spawn(function()
	while running do
		local ok, err = pcall(step)
		if not ok then
			warn("[RobScript DR] " .. tostring(err))
		end
		task.wait(0.05)
	end
end)

----- Interface -----

local Window = Airflow:CreateWindow({
	Name = "Robscript Team",
	LoadingSubtitle = "defeat ragdolls - auto farm",
	ToggleUIKeybind = "RightControl",
	Size = UDim2.fromOffset(760, 520),
	Theme = ThemeValue,
	Density = "Compact",
	Backdrop = { Weather = "Fireflies", Tint = 0.35 },
	ConfigurationSaving = { Enabled = true, FolderName = "RobScriptDR" },
	Home = {
		Title = "Robscript Team",
		Welcome = "Welcome back,",
		Tier = "FARM 1.0",
		Stats = { "Players", "Session", "FPS", "Ping", "Game", "Executor" },
	},
})

local FarmTab = Window:CreateTab({ Name = "Farm", Icon = "swords" })
local FarmBox = FarmTab:CreateGroupbox({ Name = "Punch", Side = "Left" })
local FarmStatusBox = FarmTab:CreateGroupbox({ Name = "Session", Side = "Right" })

FarmBox:CreateToggle({
	Name = "Auto Farm",
	Desc = "Punch the nearest NPC until it drops its coins",
	Flag = "AutoFarm",
	CurrentValue = false,
})
FarmBox:CreateSlider({
	Name = "Search radius",
	Range = { 50, 500 },
	Increment = 10,
	Suffix = " studs",
	CurrentValue = 200,
	Flag = "FarmRadius",
})
FarmBox:CreateSlider({
	Name = "Punch range",
	Range = { 3, 8 },
	Increment = 0.5,
	Suffix = " studs",
	CurrentValue = 4.5,
	Flag = "PunchRange",
})
FarmBox:CreateSlider({
	Name = "Punch delay",
	Range = { 0.3, 1 },
	Increment = 0.05,
	Suffix = " s",
	CurrentValue = 0.45,
	Flag = "PunchDelay",
})
FarmBox:CreateToggle({
	Name = "Stick to target",
	Desc = "Hold the character next to the NPC instead of stepping on every punch",
	Flag = "StickToTarget",
	CurrentValue = true,
	DependsOn = "AutoFarm",
})
FarmBox:CreateSlider({
	Name = "Stick distance",
	Range = { 2, 6 },
	Increment = 0.5,
	Suffix = " studs",
	CurrentValue = 3.5,
	Flag = "StickDistance",
	DependsOn = "AutoFarm",
})
FarmBox:CreateToggle({
	Name = "Teleport to target",
	Desc = "Jump to an NPC that is further than 40 studs away",
	Flag = "TeleportToTarget",
	CurrentValue = true,
	DependsOn = "AutoFarm",
})

FarmStatusBox:CreateStatus({
	Name = "Target",
	Style = "Row",
	UpdateRate = 0.5,
	Update = function()
		return string.format("<Success>%s</Success> <Muted>%d hp</Muted>", state.target, state.targetHp)
	end,
})
FarmStatusBox:CreateStatus({
	Name = "Punches",
	Style = "Row",
	UpdateRate = 0.5,
	Update = function()
		return string.format("<Success>%d</Success>", state.punches)
	end,
})
FarmStatusBox:CreateStatus({
	Name = "Knockouts",
	Style = "Row",
	UpdateRate = 0.5,
	Update = function()
		return string.format("<Success>%d</Success>", state.kills)
	end,
})

local CoinsTab = Window:CreateTab({ Name = "Drops", Icon = "coins" })
local CoinsBox = CoinsTab:CreateGroupbox({ Name = "Collector", Side = "Left" })
local CoinsStatusBox = CoinsTab:CreateGroupbox({ Name = "Drops", Side = "Right" })

CoinsBox:CreateToggle({
	Name = "Auto Collect",
	Desc = "Claim every token coin and drumstick the NPCs drop",
	Flag = "AutoCollect",
	CurrentValue = true,
})
CoinsBox:CreateSlider({
	Name = "Claim radius",
	Range = { 3, 10 },
	Increment = 0.5,
	Suffix = " studs",
	CurrentValue = 5.5,
	Flag = "ClaimRadius",
})
CoinsBox:CreateSlider({
	Name = "Search radius",
	Desc = "Drops further than this are left alone",
	Range = { 20, 200 },
	Increment = 10,
	Suffix = " studs",
	CurrentValue = 60,
	Flag = "CollectRadius",
})
CoinsBox:CreateToggle({
	Name = "Teleport to drops",
	Desc = "Step to a drop that is out of claim range",
	Flag = "TeleportToDrops",
	CurrentValue = true,
	DependsOn = "AutoCollect",
})
CoinsBox:CreateButton({
	Name = "Collect now",
	Desc = "Claim every drop in range right away",
	Callback = function()
		lastClaim = 0
		settleUntil = 0
		collectDrops()
	end,
})

CoinsStatusBox:CreateStatus({
	Name = "Claimed",
	Style = "Row",
	UpdateRate = 0.5,
	Update = function()
		return string.format("<Success>%d</Success>", state.claims)
	end,
})
CoinsStatusBox:CreateStatus({
	Name = "On ground",
	Style = "Row",
	UpdateRate = 0.5,
	Update = function()
		local coins = Workspace:FindFirstChild("TokenCoins")
		local food = Workspace:FindFirstChild("Drumsticks")
		return string.format("<Success>%d</Success> <Muted>coins</Muted> <Success>%d</Success> <Muted>food</Muted>",
			coins and #coins:GetChildren() or 0,
			food and #food:GetChildren() or 0)
	end,
})

local SettingsTab = Window:CreateTab({ Name = "Settings", Icon = "settings" })
local SettingsBox = SettingsTab:CreateGroupbox({ Name = "Script", Side = "Left" })

SettingsBox:CreateStatus({
	Name = "Game",
	Style = "Row",
	UpdateRate = 2,
	Update = function()
		return string.format("<Muted>%s</Muted>", game.Name)
	end,
})
SettingsBox:CreateStatus({
	Name = "Remotes",
	Style = "Row",
	UpdateRate = 2,
	Update = function()
		if Ready then
			return "<Success>ready</Success>"
		end
		return "<Error>missing</Error>"
	end,
})
SettingsBox:CreateButton({
	Name = "Unload",
	Desc = "Close the interface and stop everything",
	Style = "Primary",
	Callback = function()
		local api = getgenv().RobScriptDR
		if type(api) == "table" and type(api.Destroy) == "function" then
			api.Destroy()
		end
	end,
})

----- Teardown -----

local function destroy()
	if not running then
		return
	end
	running = false
	pcall(function()
		glueConnection:Disconnect()
	end)
	pcall(function()
		Window:Destroy()
	end)
	Airflow = nil
	if getgenv().RobScriptDR then
		getgenv().RobScriptDR = nil
	end
end

----- Public interface -----

getgenv().RobScriptDR = {
	Version = "1.0",
	Actions = {
		Collect = function()
			lastClaim = 0
			settleUntil = 0
			collectDrops()
		end,
		Destroy = destroy,
	},
	GetStats = function()
		return {
			punches = state.punches,
			kills = state.kills,
			claims = state.claims,
			target = state.target,
			targetHp = state.targetHp,
			coins = #(Workspace:FindFirstChild("TokenCoins") and Workspace.TokenCoins:GetChildren() or {}),
			food = #(Workspace:FindFirstChild("Drumsticks") and Workspace.Drumsticks:GetChildren() or {}),
		}
	end,
	GetNearby = function()
		local root = getRoot()
		local list = {}
		if not root then
			return list
		end
		for _, holder in ipairs(refreshContainers()) do
			for _, model in ipairs(holder:GetChildren()) do
				if isFarmable(model) then
					local part = getHitPart(model)
					if part then
						list[#list + 1] = {
							name = model.Name,
							health = math.floor(model:GetAttribute("Health") or 0),
							distance = math.floor((part.Position - root.Position).Magnitude),
						}
					end
				end
			end
		end
		return list
	end,
	Get = function(name, fallback)
		return flag(name, fallback)
	end,
	Set = function(name, value)
		return setFlag(name, value)
	end,
	Destroy = destroy,
}

----- Boot -----

pcall(function()
	Window:LoadAutoload()
end)

if Ready then
	Window:Notify({ Title = "Robscript Team", Content = "Defeat Ragdolls Auto Farm is ready", Type = "Success" })
else
	Window:Notify({ Title = "Robscript Team", Content = "Remotes not found, this build is not supported", Type = "Error" })
end

return getgenv().RobScriptDR

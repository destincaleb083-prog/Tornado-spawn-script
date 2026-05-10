-- TornadoSystem.lua
-- Place in ServerScriptService

local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")

-- ========================
-- SETTINGS
-- ========================
local Config = {
	SpawnHeight = 300,         -- How high the tornado starts
	TouchdownTime = 4,         -- Seconds to touch down
	MoveSpeed = 18,            -- Base movement speed
	WanderStrength = 60,       -- How unpredictable the path is
	NoiseScale = 0.3,          -- Perlin noise scale (lower = smoother turns)
	PullRadius = 80,           -- Radius to pull players
	DamageRadius = 20,         -- Radius to deal damage
	DamageAmount = 10,         -- Damage per second inside damage radius
	FunnelSegments = 12,       -- Number of funnel parts
	FunnelHeight = 120,        -- Total height of funnel
	LifeTime = 60,             -- How long tornado lasts (seconds)
	DebrisCount = 20,          -- Number of debris pieces
}

-- ========================
-- FUNNEL BUILDER
-- ========================
local function buildFunnel(root)
	local funnelParts = {}
	local folder = Instance.new("Folder")
	folder.Name = "TornadoFunnel"
	folder.Parent = root

	for i = 1, Config.FunnelSegments do
		local t = i / Config.FunnelSegments
		local size = Vector3.new(
			math.lerp(2, 30, t),   -- narrow at bottom, wide at top
			Config.FunnelHeight / Config.FunnelSegments + 1,
			math.lerp(2, 30, t)
		)

		local part = Instance.new("Part")
		part.Size = size
		part.Shape = Enum.PartType.Cylinder
		part.CFrame = root.CFrame * CFrame.new(0, i * (Config.FunnelHeight / Config.FunnelSegments), 0)
		part.Anchored = true
		part.CanCollide = false
		part.CastShadow = false
		part.Transparency = math.lerp(0.3, 0.85, t)
		part.Color = Color3.fromRGB(120, 100, 80)
		part.Material = Enum.Material.SmoothPlastic
		part.Parent = folder

		-- Add rotation tween for spinning effect
		local spinSpeed = math.lerp(8, 3, t) -- faster at bottom
		local spinTween = TweenService:Create(part, TweenInfo.new(
			spinSpeed,
			Enum.EasingStyle.Linear,
			Enum.EasingDirection.InOut,
			-1 -- infinite
		), {CFrame = part.CFrame * CFrame.Angles(0, math.pi * 2, 0)})
		spinTween:Play()

		table.insert(funnelParts, {part = part, index = i})
	end

	return funnelParts, folder
end

-- ========================
-- DEBRIS SYSTEM
-- ========================
local function spawnDebris(tornadoRoot)
	local debrisList = {}

	for i = 1, Config.DebrisCount do
		local debris = Instance.new("Part")
		debris.Size = Vector3.new(
			math.random(1, 4),
			math.random(1, 4),
			math.random(1, 4)
		)
		debris.Color = Color3.fromRGB(
			math.random(80, 140),
			math.random(60, 100),
			math.random(40, 80)
		)
		debris.Material = Enum.Material.SmoothPlastic
		debris.Anchored = true
		debris.CanCollide = false
		debris.CastShadow = false

		local angle = math.random() * math.pi * 2
		local radius = math.random(10, 40)
		local height = math.random(5, Config.FunnelHeight - 10)

		debris.CFrame = tornadoRoot.CFrame * CFrame.new(
			math.cos(angle) * radius,
			height,
			math.sin(angle) * radius
		)
		debris.Parent = workspace

		table.insert(debrisList, {
			part = debris,
			angle = angle,
			radius = radius,
			height = height,
			orbitSpeed = math.random(2, 5) + math.random() * 2
		})
	end

	return debrisList
end

-- ========================
-- MAIN TORNADO SPAWNER
-- ========================
local function spawnTornado(position)
	-- Root part (invisible anchor)
	local root = Instance.new("Part")
	root.Size = Vector3.new(1, 1, 1)
	root.Anchored = true
	root.CanCollide = false
	root.Transparency = 1
	root.CFrame = CFrame.new(position.X, Config.SpawnHeight, position.Z)
	root.Parent = workspace

	-- Build funnel
	local funnelParts, funnelFolder = buildFunnel(root)
	funnelFolder.Parent = workspace

	-- Spawn debris
	local debrisList = spawnDebris(root)

	-- Ground dust effect
	local dust = Instance.new("Part")
	dust.Size = Vector3.new(20, 2, 20)
	dust.Shape = Enum.PartType.Cylinder
	dust.Anchored = true
	dust.CanCollide = false
	dust.Transparency = 0.5
	dust.Color = Color3.fromRGB(150, 120, 90)
	dust.CFrame = CFrame.new(position.X, position.Y, position.Z)
	dust.Parent = workspace

	-- ========================
	-- TOUCHDOWN ANIMATION
	-- ========================
	local touchdownComplete = false
	local targetY = position.Y

	local touchdownTween = TweenService:Create(root, TweenInfo.new(
		Config.TouchdownTime,
		Enum.EasingStyle.Quad,
		Enum.EasingDirection.In
	), {CFrame = CFrame.new(position.X, targetY, position.Z)})

	touchdownTween:Play()
	touchdownTween.Completed:Connect(function()
		touchdownComplete = true
	end)

	-- ========================
	-- MOVEMENT & GAME LOOP
	-- ========================
	local elapsed = 0
	local noiseOffset = math.random(0, 1000)
	local currentPos = Vector3.new(position.X, targetY, position.Z)

	local connection
	connection = RunService.Heartbeat:Connect(function(dt)
		elapsed += dt

		-- Stop after lifetime
		if elapsed >= Config.LifeTime then
			connection:Disconnect()
			-- Cleanup
			funnelFolder:Destroy()
			root:Destroy()
			dust:Destroy()
			for _, d in debrisList do
				d.part:Destroy()
			end
			return
		end

		-- Unpredictable movement using Perlin noise
		local noiseX = math.noise(elapsed * Config.NoiseScale + noiseOffset, 0) * Config.WanderStrength
		local noiseZ = math.noise(0, elapsed * Config.NoiseScale + noiseOffset) * Config.WanderStrength

		local moveDir = Vector3.new(noiseX, 0, noiseZ).Unit
		currentPos = currentPos + moveDir * Config.MoveSpeed * dt

		-- Clamp Y to ground during touchdown
		local rootY = touchdownComplete and targetY or root.CFrame.Y
		root.CFrame = CFrame.new(currentPos.X, rootY, currentPos.Z)

		-- Update funnel segment positions
		for _, seg in funnelParts do
			local t = seg.index / Config.FunnelSegments
			seg.part.CFrame = CFrame.new(
				currentPos.X,
				rootY + seg.index * (Config.FunnelHeight / Config.FunnelSegments),
				currentPos.Z
			)
		end

		-- Update dust position
		dust.CFrame = CFrame.new(currentPos.X, targetY + 1, currentPos.Z)

		-- Orbit debris
		for _, d in debrisList do
			d.angle += d.orbitSpeed * dt
			d.part.CFrame = CFrame.new(
				currentPos.X + math.cos(d.angle) * d.radius,
				rootY + d.height,
				currentPos.Z + math.sin(d.angle) * d.radius
			)
		end

		-- ========================
		-- PLAYER INTERACTIONS
		-- ========================
		for _, player in Players:GetPlayers() do
			local char = player.Character
			if not char then continue end
			local hrp = char:FindFirstChild("HumanoidRootPart")
			local hum = char:FindFirstChild("Humanoid")
			if not hrp or not hum then continue end

			local dist = (hrp.Position - Vector3.new(currentPos.X, hrp.Position.Y, currentPos.Z)).Magnitude

			-- Pull effect
			if dist < Config.PullRadius and dist > Config.DamageRadius then
				local pullDir = (Vector3.new(currentPos.X, hrp.Position.Y, currentPos.Z) - hrp.Position).Unit
				local strength = (1 - dist / Config.PullRadius) * 60
				hrp.Velocity = hrp.Velocity + pullDir * strength * dt
			end

			-- Damage + throw effect
			if dist < Config.DamageRadius then
				hum:TakeDamage(Config.DamageAmount * dt)
				-- Launch player upward
				local throwDir = Vector3.new(
					math.random(-1, 1),
					2,
					math.random(-1, 1)
				).Unit
				hrp.Velocity = throwDir * 80
			end
		end
	end)

	return root
end

-- ========================
-- SPAWN TORNADO IN WORLD
-- ========================
-- Change this Vector3 to wherever you want it to spawn
task.wait(3) -- Small delay before spawning
spawnTornado(Vector3.new(0, 3, 0))

-- ========================
-- PARTICLE FUNNEL BUILDER
-- Replaces cylinder segments with particle emitters
-- ========================
local function buildFunnel(startPos)
	local folder = Instance.new("Folder")
	folder.Name = "StudioLiteTornado"
	folder.Parent = workspace

	local funnelParts = {}

	for i = 1, Config.FunnelSegments do
		local t = i / Config.FunnelSegments
		local width = math.lerp(2, 32, t)
		local height = startPos.Y + i * (Config.FunnelHeight / Config.FunnelSegments)

		-- Invisible anchor part for each emitter ring
		local anchor = Instance.new("Part")
		anchor.Size = Vector3.new(1, 1, 1)
		anchor.Anchored = true
		anchor.CanCollide = false
		anchor.Transparency = 1
		anchor.CFrame = CFrame.new(startPos.X, height, startPos.Z)
		anchor.Parent = folder

		-- Dust particle emitter
		local emitter = Instance.new("ParticleEmitter")
		emitter.Texture = "rbxassetid://243660364"  -- Smoke/dust texture
		emitter.Color = ColorSequence.new({
			ColorSequenceKeypoint.new(0, Color3.fromRGB(120, 100, 80)),   -- dark dust
			ColorSequenceKeypoint.new(0.5, Color3.fromRGB(160, 140, 120)), -- mid dust
			ColorSequenceKeypoint.new(1, Color3.fromRGB(200, 190, 180)),  -- light dust
		})
		emitter.Transparency = NumberSequence.new({
			NumberSequenceKeypoint.new(0, 0.2),
			NumberSequenceKeypoint.new(0.5, math.lerp(0.4, 0.75, t)),
			NumberSequenceKeypoint.new(1, 1),
		})
		emitter.Size = NumberSequence.new({
			NumberSequenceKeypoint.new(0, width * 0.3),
			NumberSequenceKeypoint.new(1, width * 0.6),
		})
		emitter.Rate = math.lerp(40, 15, t)     -- More particles at bottom
		emitter.Speed = NumberRange.new(width * 0.8, width * 1.5)
		emitter.SpreadAngle = Vector2.new(180, 180) -- Full circular spread
		emitter.Lifetime = NumberRange.new(0.3, 0.7)
		emitter.RotSpeed = NumberRange.new(-200, 200)
		emitter.Rotation = NumberRange.new(0, 360)
		emitter.LightEmission = 0.05
		emitter.LightInfluence = 0.8
		emitter.LockedToPart = false
		emitter.Parent = anchor

		-- Inner swirl emitter (darker, faster)
		local innerEmitter = Instance.new("ParticleEmitter")
		innerEmitter.Texture = "rbxassetid://243660364"
		innerEmitter.Color = ColorSequence.new({
			ColorSequenceKeypoint.new(0, Color3.fromRGB(80, 65, 55)),
			ColorSequenceKeypoint.new(1, Color3.fromRGB(110, 90, 75)),
		})
		innerEmitter.Transparency = NumberSequence.new({
			NumberSequenceKeypoint.new(0, 0.1),
			NumberSequenceKeypoint.new(1, 1),
		})
		innerEmitter.Size = NumberSequence.new({
			NumberSequenceKeypoint.new(0, width * 0.1),
			NumberSequenceKeypoint.new(1, width * 0.3),
		})
		innerEmitter.Rate = math.lerp(60, 20, t)
		innerEmitter.Speed = NumberRange.new(width * 0.4, width * 0.9)
		innerEmitter.SpreadAngle = Vector2.new(180, 180)
		innerEmitter.Lifetime = NumberRange.new(0.2, 0.5)
		innerEmitter.RotSpeed = NumberRange.new(-400, 400)
		innerEmitter.Rotation = NumberRange.new(0, 360)
		innerEmitter.LightEmission = 0
		innerEmitter.LightInfluence = 1
		innerEmitter.LockedToPart = false
		innerEmitter.Parent = anchor

		table.insert(funnelParts, {part = anchor, index = i})
	end

	-- Ground dust burst at base
	local groundAnchor = Instance.new("Part")
	groundAnchor.Size = Vector3.new(1, 1, 1)
	groundAnchor.Anchored = true
	groundAnchor.CanCollide = false
	groundAnchor.Transparency = 1
	groundAnchor.CFrame = CFrame.new(startPos.X, startPos.Y + 2, startPos.Z)
	groundAnchor.Parent = folder

	local groundDust = Instance.new("ParticleEmitter")
	groundDust.Texture = "rbxassetid://243660364"
	groundDust.Color = ColorSequence.new({
		ColorSequenceKeypoint.new(0, Color3.fromRGB(140, 115, 90)),
		ColorSequenceKeypoint.new(1, Color3.fromRGB(180, 160, 140)),
	})
	groundDust.Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 0.1),
		NumberSequenceKeypoint.new(0.6, 0.5),
		NumberSequenceKeypoint.new(1, 1),
	})
	groundDust.Size = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 8),
		NumberSequenceKeypoint.new(1, 20),
	})
	groundDust.Rate = 80
	groundDust.Speed = NumberRange.new(20, 40)
	groundDust.SpreadAngle = Vector2.new(180, 30)
	groundDust.Lifetime = NumberRange.new(0.8, 1.5)
	groundDust.RotSpeed = NumberRange.new(-150, 150)
	groundDust.Rotation = NumberRange.new(0, 360)
	groundDust.Parent = groundAnchor

	table.insert(funnelParts, {part = groundAnchor, index = 0})

	return funnelParts, folder
end

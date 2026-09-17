----------------------------------------------------------------
--// 3D ANIMATION VIEWER
----------------------------------------------------------------

local Viewport = Instance.new("ViewportFrame")
Viewport.Name = "AnimationViewport"
Viewport.Size = UDim2.fromOffset(525, 245)
Viewport.Position = UDim2.fromOffset(10, 105)
Viewport.BackgroundColor3 = Color3.fromRGB(9, 9, 13)
Viewport.BorderSizePixel = 0
Viewport.Ambient = Color3.fromRGB(190, 190, 190)
Viewport.LightColor = Color3.fromRGB(255, 255, 255)
Viewport.LightDirection = Vector3.new(-1, -1, -1)
Viewport.Parent = Details

Instance.new("UICorner", Viewport).CornerRadius = UDim.new(0, 10)

local WorldModel = Instance.new("WorldModel")
WorldModel.Parent = Viewport

local Camera = Instance.new("Camera")
Camera.FieldOfView = 45
Camera.Parent = Viewport
Viewport.CurrentCamera = Camera

----------------------------------------------------------------
--// VIEWPORT FLOOR
----------------------------------------------------------------

local Floor = Instance.new("Part")
Floor.Name = "Floor"
Floor.Size = Vector3.new(20, 0.2, 20)
Floor.Anchored = true
Floor.CanCollide = false
Floor.Material = Enum.Material.SmoothPlastic
Floor.Color = Color3.fromRGB(25, 25, 31)
Floor.Position = Vector3.new(0, -3, 0)
Floor.Parent = WorldModel

----------------------------------------------------------------
--// VIEWPORT LIGHT
----------------------------------------------------------------

local LightPart = Instance.new("Part")
LightPart.Name = "Light"
LightPart.Size = Vector3.new(1, 1, 1)
LightPart.Transparency = 1
LightPart.Anchored = true
LightPart.CanCollide = false
LightPart.Position = Vector3.new(0, 8, 4)
LightPart.Parent = WorldModel

local Light = Instance.new("PointLight")
Light.Brightness = 2
Light.Range = 30
Light.Shadows = true
Light.Parent = LightPart

----------------------------------------------------------------
--// VIEWPORT STATE
----------------------------------------------------------------

local ReplayCharacter = nil
local ReplayHumanoid = nil
local ReplayAnimator = nil
local ReplayTrack = nil

local replayTime = 0
local replayPlaying = false

----------------------------------------------------------------
--// CLEAN VIEWPORT
----------------------------------------------------------------

local function clearReplay()
	if ReplayTrack then
		pcall(function()
			ReplayTrack:Stop()
			ReplayTrack:Destroy()
		end)
	end

	ReplayTrack = nil
	ReplayHumanoid = nil
	ReplayAnimator = nil

	if ReplayCharacter then
		ReplayCharacter:Destroy()
	end

	ReplayCharacter = nil
end

----------------------------------------------------------------
--// PREPARE CHARACTER
----------------------------------------------------------------

local function prepareCharacter(character)
	for _, obj in ipairs(character:GetDescendants()) do

		if obj:IsA("Script")
			or obj:IsA("LocalScript")
			or obj:IsA("ModuleScript") then

			obj:Destroy()

		elseif obj:IsA("BasePart") then

			obj.Anchored = false
			obj.CanCollide = false
			obj.CanTouch = false
			obj.CanQuery = false

		elseif obj:IsA("Tool") then
			obj:Destroy()
		end
	end

	local root = character:FindFirstChild("HumanoidRootPart")

	if root then
		root.Anchored = false
	end

	return character
end

----------------------------------------------------------------
--// CAMERA
----------------------------------------------------------------

local function positionCamera(character)
	local root = character:FindFirstChild("HumanoidRootPart")

	if not root then
		return
	end

	local position = root.Position

	Camera.CFrame = CFrame.lookAt(
		position + Vector3.new(0, 1.5, 8),
		position + Vector3.new(0, 1.5, 0)
	)
end

----------------------------------------------------------------
--// LOAD ANIMATION INTO VIEWPORT
----------------------------------------------------------------

local function loadReplay(data)
	clearReplay()

	if not data then
		return
	end

	local originalPlayer = Players:FindFirstChild(data.player)

	if not originalPlayer then
		Status.Text = "Original player is no longer in the server."
		return
	end

	local originalCharacter = originalPlayer.Character

	if not originalCharacter then
		Status.Text = "Original character no longer exists."
		return
	end

	----------------------------------------------------------------
	-- CLONE
	----------------------------------------------------------------

	local clone

	local success, result = pcall(function()
		return originalCharacter:Clone()
	end)

	if not success or not result then
		Status.Text = "Could not clone enemy character."
		return
	end

	clone = result

	prepareCharacter(clone)

	clone.Name = "Replay_" .. data.id
	clone.Parent = WorldModel

	ReplayCharacter = clone

	----------------------------------------------------------------
	-- ROOT
	----------------------------------------------------------------

	local root = clone:FindFirstChild("HumanoidRootPart")

	if not root then
		clearReplay()
		Status.Text = "Replay character has no HumanoidRootPart."
		return
	end

	clone:PivotTo(
		CFrame.new(0, -2.9, 0)
	)

	----------------------------------------------------------------
	-- HUMANOID
	----------------------------------------------------------------

	local humanoid = clone:FindFirstChildOfClass("Humanoid")

	if not humanoid then
		clearReplay()
		Status.Text = "Replay character has no Humanoid."
		return
	end

	ReplayHumanoid = humanoid

	----------------------------------------------------------------
	-- ANIMATOR
	----------------------------------------------------------------

	local animator =
		humanoid:FindFirstChildOfClass("Animator")

	if not animator then
		animator = Instance.new("Animator")
		animator.Parent = humanoid
	end

	ReplayAnimator = animator

	----------------------------------------------------------------
	-- ANIMATION
	----------------------------------------------------------------

	local animation = Instance.new("Animation")

	animation.AnimationId =
		"rbxassetid://" .. data.id

	local successTrack, track = pcall(function()
		return animator:LoadAnimation(animation)
	end)

	animation:Destroy()

	if not successTrack or not track then
		clearReplay()

		Status.Text =
			"Could not load animation.\n" ..
			"ID: " .. data.id

		return
	end

	ReplayTrack = track

	ReplayTrack.Looped = false
	ReplayTrack.Priority = Enum.AnimationPriority.Action

	----------------------------------------------------------------
	-- DURATION
	----------------------------------------------------------------

	local timeout = os.clock() + 3

	while ReplayTrack.Length <= 0 and os.clock() < timeout do
		RunService.Heartbeat:Wait()
	end

	data.duration = ReplayTrack.Length

	----------------------------------------------------------------
	-- CAMERA
	----------------------------------------------------------------

	positionCamera(clone)

	----------------------------------------------------------------
	-- START
	----------------------------------------------------------------

	replayTime = 0
	currentTime = 0
	currentFrame = 0

	ReplayTrack:Play(
		0,
		1,
		0
	)

	ReplayTrack.TimePosition = 0

	replayPlaying = false
	playing = false

	Play.Text = "▶ PLAY"

	TimeLabel.Text =
		string.format(
			"0.000s / %.3fs  |  Frame 0",
			data.duration
		)

	DelayLabel.Text =
		string.format(
			"Saved delay: %.3fs",
			data.delay
		)

	Status.Text =
		"REPLAY READY\n" ..
		data.name ..
		"\nAnimation ID: " .. data.id
end

----------------------------------------------------------------
--// SET REPLAY TIME
----------------------------------------------------------------

local function setReplayTime(time)
	if not ReplayTrack then
		return
	end

	local data = selectedID and Animations[selectedID]

	if not data then
		return
	end

	local duration = ReplayTrack.Length

	if duration <= 0 then
		return
	end

	replayTime = math.clamp(
		time,
		0,
		duration
	)

	currentTime = replayTime

	currentFrame =
		math.floor(
			replayTime * FPS + 0.5
		)

	ReplayTrack.TimePosition = replayTime

	TimeLabel.Text =
		string.format(
			"%.3fs / %.3fs  |  Frame %d",
			replayTime,
			duration,
			currentFrame
	)

	local alpha =
		math.clamp(
			replayTime / duration,
			0,
			1
		)

	TimelineMarker.Position =
		UDim2.new(
			alpha,
			-1,
			0,
			0
		)
end

----------------------------------------------------------------
--// PLAY
----------------------------------------------------------------

Play.MouseButton1Click:Connect(function()

	if not ReplayTrack then
		return
	end

	replayPlaying = not replayPlaying
	playing = replayPlaying

	if replayPlaying then

		ReplayTrack:Play(
			0,
			1,
			1
		)

		ReplayTrack.TimePosition =
			replayTime

		Play.Text = "⏸ PAUSE"

	else

		ReplayTrack:AdjustSpeed(0)

		Play.Text = "▶ PLAY"
	end
end)

----------------------------------------------------------------
--// PREVIOUS FRAME
----------------------------------------------------------------

Previous.MouseButton1Click:Connect(function()

	if not ReplayTrack then
		return
	end

	replayPlaying = false
	playing = false

	Play.Text = "▶ PLAY"

	setReplayTime(
		replayTime - FRAME_TIME
	)
end)

----------------------------------------------------------------
--// NEXT FRAME
----------------------------------------------------------------

Next.MouseButton1Click:Connect(function()

	if not ReplayTrack then
		return
	end

	replayPlaying = false
	playing = false

	Play.Text = "▶ PLAY"

	setReplayTime(
		replayTime + FRAME_TIME
	)
end)

----------------------------------------------------------------
--// RESET
----------------------------------------------------------------

Reset.MouseButton1Click:Connect(function()

	if not ReplayTrack then
		return
	end

	replayPlaying = false
	playing = false

	Play.Text = "▶ PLAY"

	setReplayTime(0)
end)

----------------------------------------------------------------
--// SELECT ANIMATION
----------------------------------------------------------------

local oldSelectAnimation = selectAnimation

selectAnimation = function(id)

	local data = Animations[id]

	if not data then
		return
	end

	selectedID = id

	SelectedLabel.Text =
		data.name .. "  |  " .. data.id

	DelayLabel.Text =
		string.format(
			"Saved delay: %.3fs",
			data.delay
		)

	Arm.Text =
		data.parry
		and "ARM: ON"
		or "ARM: OFF"

	Status.Text =
		"Loading 3D replay..."

	loadReplay(data)
end

----------------------------------------------------------------
--// LIVE REPLAY UPDATE
----------------------------------------------------------------

RunService.RenderStepped:Connect(function(dt)

	if not ReplayTrack then
		return
	end

	if not replayPlaying then
		return
	end

	local duration = ReplayTrack.Length

	if duration <= 0 then
		return
	end

	replayTime =
		ReplayTrack.TimePosition

	currentTime = replayTime

	currentFrame =
		math.floor(
			replayTime * FPS + 0.5
		)

	TimeLabel.Text =
		string.format(
			"%.3fs / %.3fs  |  Frame %d",
			replayTime,
			duration,
			currentFrame
		)

	local alpha =
		math.clamp(
			replayTime / duration,
			0,
			1
		)

	TimelineMarker.Position =
		UDim2.new(
			alpha,
			-1,
			0,
			0
		)

	if replayTime >= duration - 0.01 then

		replayPlaying = false
		playing = false

		Play.Text = "▶ PLAY"

		ReplayTrack:AdjustSpeed(0)
	end
end)

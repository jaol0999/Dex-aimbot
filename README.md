local OrionLib = loadstring(game:HttpGet(('https://raw.githubusercontent.com/jensonhirst/Orion/main/source')))()

local Window = OrionLib:MakeWindow({
	Name = "<font color='rgb(255,0,0)'>Dex</font> hub",
	HidePremium = false,
	SaveConfig = true,
	ConfigFolder = "OrionTest"
})

-- ABA AIMBOT
local Tab = Window:MakeTab({
	Name = "aimbot",
	Icon = "shield-check",
	PremiumOnly = false
})

local aimbotAtivo = false
local fovCircle = nil
local teamCheckAimbot = false

local function criarFOV()
	if fovCircle then fovCircle:Remove() end
	local camera = workspace.CurrentCamera
	local viewportSize = camera.ViewportSize
	local raio = math.min(viewportSize.X, viewportSize.Y) * 0.2 / 2
	local Drawing = Drawing or getgenv().Drawing
	fovCircle = Drawing.new("Circle")
	fovCircle.Position = Vector2.new(viewportSize.X / 2, viewportSize.Y / 2)
	fovCircle.Radius = raio
	fovCircle.Thickness = 2
	fovCircle.Transparency = 1
	fovCircle.Color = Color3.fromRGB(255,255,255)
	fovCircle.Filled = false
	fovCircle.Visible = true
end

local function removerFOV()
	if fovCircle then fovCircle:Remove() fovCircle = nil end
end

local function getTargetPart(character)
	-- Tenta achar "Head" ou a parte mais alta do personagem
	if character:FindFirstChild("Head") then
		return character.Head
	end
	local highest = nil
	for _, part in ipairs(character:GetChildren()) do
		if part:IsA("BasePart") then
			if not highest or part.Position.Y > highest.Position.Y then
				highest = part
			end
		end
	end
	return highest
end

local function isVisible(origin, target, targetPart)
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Blacklist
	params.FilterDescendantsInstances = {game.Players.LocalPlayer.Character}
	params.IgnoreWater = true
	local result = workspace:Raycast(origin, (target - origin).Unit * (target - origin).Magnitude, params)
	return not result or result.Instance == targetPart
end

local function getNearestVisiblePlayerInFOV()
	local players = game:GetService("Players")
	local localPlayer = players.LocalPlayer
	local camera = workspace.CurrentCamera
	local viewportSize = camera.ViewportSize
	local center = Vector2.new(viewportSize.X / 2, viewportSize.Y / 2)
	local fov = math.min(viewportSize.X, viewportSize.Y) * 0.2 / 2
	local nearest, shortest = nil, math.huge

	for _, player in ipairs(players:GetPlayers()) do
		if player ~= localPlayer and player.Character then
			local targetPart = getTargetPart(player.Character)
			if targetPart and player.Character:FindFirstChildOfClass("Humanoid") and player.Character:FindFirstChildOfClass("Humanoid").Health > 0 then
				if teamCheckAimbot and player.Team == localPlayer.Team then
					continue
				end
				local screenPos, onscreen = camera:WorldToViewportPoint(targetPart.Position)
				if onscreen then
					local dist = (Vector2.new(screenPos.X, screenPos.Y) - center).Magnitude
					if dist <= fov and dist < shortest and isVisible(camera.CFrame.Position, targetPart.Position, targetPart) then
						nearest = targetPart
						shortest = dist
					end
				end
			end
		end
	end
	return nearest
end

local function aimbotFunc()
	local camera = workspace.CurrentCamera
	while aimbotAtivo do
		local targetPart = getNearestVisiblePlayerInFOV()
		if targetPart then
			camera.CFrame = CFrame.new(camera.CFrame.Position, targetPart.Position)
		end
		task.wait()
	end
end

Tab:AddToggle({
	Name = "ligar aimbot",
	Default = false,
	Callback = function(Value)
		aimbotAtivo = Value
		if aimbotAtivo then
			criarFOV()
			aimbotFunc()
		else
			removerFOV()
		end
	end    
})

Tab:AddToggle({
	Name = "team check",
	Default = false,
	Callback = function(Value)
		teamCheckAimbot = Value
	end    
})

-- ABA ESP¿
local EspTab = Window:MakeTab({
	Name = "Esp¿",
	Icon = "shield-check",
	PremiumOnly = false
})

local chamsEnabled = false
local teamCheck_ESP = false
local espObjects = {}

local function removeChams()
	for player, adorns in pairs(espObjects) do
		for _, adorn in ipairs(adorns) do
			if adorn and adorn.Parent then adorn:Destroy() end
		end
	end
	espObjects = {}
end

local function createChamsForPlayer(player)
	local localPlayer = game.Players.LocalPlayer
	if player == localPlayer then return end -- não mostra você
	if not player.Character then return end
	if teamCheck_ESP and player.Team == localPlayer.Team then return end -- team check ESP
	espObjects[player] = espObjects[player] or {}
	for _, part in ipairs(player.Character:GetChildren()) do
		if part:IsA("BasePart") then
			local cham = Instance.new("BoxHandleAdornment")
			cham.Name = "ESPCham"
			cham.Size = part.Size
			cham.Adornee = part
			cham.AlwaysOnTop = true
			cham.ZIndex = 5
			cham.Color3 = Color3.fromRGB(0,255,255)
			cham.Transparency = 0.25
			cham.Parent = part
			table.insert(espObjects[player], cham)
		end
	end
end

local function removeChamsForPlayer(player)
	if espObjects[player] then
		for _, adorn in ipairs(espObjects[player]) do
			if adorn and adorn.Parent then adorn:Destroy() end
		end
		espObjects[player] = nil
	end
end

local function refreshChams()
	removeChams()
	local players = game:GetService("Players")
	local localPlayer = players.LocalPlayer
	for _, player in ipairs(players:GetPlayers()) do
		if player ~= localPlayer and player.Character and player.Character:FindFirstChild("Head") and player.Character:FindFirstChildOfClass("Humanoid") then
			if player.Character:FindFirstChildOfClass("Humanoid").Health > 0 then
				createChamsForPlayer(player)
			end
		end
	end
end

local function setupChams()
	local players = game:GetService("Players")
	local localPlayer = players.LocalPlayer

	for _, player in ipairs(players:GetPlayers()) do
		if player ~= localPlayer then
			player.CharacterAdded:Connect(function()
				if chamsEnabled then
					task.wait(0.5)
					createChamsForPlayer(player)
				end
			end)
			player.CharacterRemoving:Connect(function()
				removeChamsForPlayer(player)
			end)
		end
	end

	players.PlayerAdded:Connect(function(player)
		if player ~= localPlayer then
			player.CharacterAdded:Connect(function()
				if chamsEnabled then
					task.wait(0.5)
					createChamsForPlayer(player)
				end
			end)
			player.CharacterRemoving:Connect(function()
				removeChamsForPlayer(player)
			end)
		end
	end)
end

EspTab:AddToggle({
	Name = "esp-holograma",
	Default = false,
	Callback = function(Value)
		chamsEnabled = Value
		if chamsEnabled then
			setupChams()
			refreshChams()
		else
			removeChams()
		end
	end    
})

EspTab:AddToggle({
	Name = "team check",
	Default = false,
	Callback = function(Value)
		teamCheck_ESP = Value
		if chamsEnabled then
			refreshChams()
		end
	end    
})

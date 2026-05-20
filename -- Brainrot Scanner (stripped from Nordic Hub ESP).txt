-- Brainrot Scanner (stripped from Nordic Hub ESP)
-- Pure detection only: scans plots via Synchronizer, filters for target brainrots
-- Reports hits to API backend, then server hops

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local HttpService = game:GetService("HttpService")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TeleportService = game:GetService("TeleportService")

local player = Players.LocalPlayer

-- ═══ CONFIGURATION ═══
local API_ENDPOINT = "http://192.168.8.129:3000/api/hits/bulk"
local API_SECRET = "joinerhaha"
local BOT_NAME = "BOT_1"
local SCAN_DURATION = 30
local HOP_ENABLED = true

-- ═══ TARGET BRAINROTS ═══
-- Only report these. Lookup table for O(1) checks.
local TARGET_BRAINROTS = {}
local TARGET_LIST = {
	"Cerberus", "Fragrama and Chocrama", "Hydra Bunny", "Griffin",
	"Bunny and Eggy", "Sammyni Fattini", "Abyssaloco", "Meowl",
	"Rubrikiko", "Guest 666", "Sammyni Cakini", "Signore Carapace",
	"Duggy Bros", "Globa Steppa", "Garama and Madundung", "Reinito Sleighito",
	"Dragon Gingerini", "Quackini Snackini", "Money Money Reindeer",
	"Pancake and Syrup", "Arcadragon", "Fishino Clownino", "Eviledon",
	"La Lucky Grande", "La Extinct Grande", "Dragon Cannelloni",
	"Popcuru and Fizzuru", "La Casa Boo", "Los Hackers", "Fragola La La La",
	"La Secret Combinasion", "Gym Bros", "Headless Horseman", "Money Money Bros",
	"Tralaledon", "Cash or Card", "La Supreme Combinasion", "Elefanto Frigo",
	"Spyder Elephant", "Ginger Gerat", "Spooky and Pumpky",
	"Hydra Dragon Cannelloni", "Digi Narwhal", "Cooki and Milki",
	"Nacho Spyder", "Dug dug dug", "La Easter Grande", "Love Love Bear",
	"La Romantic Grande", "La Food Combinasion", "Celestial Pegasus",
	"Capitano Moby", "La Anniversary Grande", "Rosey and Teddy",
	"La Jolly Grande", "Ketupat Bros", "Los Chillis", "Foxini Lanternini",
	"Festive 67", "Chillin Chili", "Celularcini Viciosini", "Orcaledon",
	"La Taco Combinasion", "John Pork", "Skibidi Toilet", "Chipso and Queso",
	"Strawberry Elephant", "Fortunu and Cashuru", "Antonio", "La Spooky Grande",
}
for _, name in ipairs(TARGET_LIST) do
	TARGET_BRAINROTS[name] = true
end

-- ═══ GAME MODULES ═══
local Synchronizer, AnimalsData, AnimalsShared, NumberUtils
pcall(function()
	local Packages = ReplicatedStorage:FindFirstChild("Packages")
	local Datas = ReplicatedStorage:FindFirstChild("Datas")
	local Shared = ReplicatedStorage:FindFirstChild("Shared")
	local Utils = ReplicatedStorage:FindFirstChild("Utils")
	if Packages then Synchronizer = require(Packages:FindFirstChild("Synchronizer")) end
	if Datas then AnimalsData = require(Datas:FindFirstChild("Animals")) end
	if Shared then AnimalsShared = require(Shared:FindFirstChild("Animals")) end
	if Utils then NumberUtils = require(Utils:FindFirstChild("NumberUtils")) end
end)

-- ═══ SCANNER STATE ═══
local allAnimalsCache = {}
local plotChannels = {}
local lastAnimalData = {}
local pendingHits = {}  -- batch hits to send

-- ═══ MPS PARSER ═══
local function parseMPS(text)
	if not text then return 0 end
	local num, suffix = text:match("([%d%.]+)%s*([KMBT]?)")
	if not num then return 0 end
	num = tonumber(num) or 0
	if suffix == "T" then num = num * 1e12
	elseif suffix == "B" then num = num * 1e9
	elseif suffix == "M" then num = num * 1e6
	elseif suffix == "K" then num = num * 1e3 end
	return num
end

-- ═══ OVERHEAD MPS FALLBACK ═══
local function getMPSFromOverhead(plotName, slotNum)
	local plots = Workspace:FindFirstChild("Plots")
	if not plots then return nil end
	local plot = plots:FindFirstChild(plotName)
	if not plot then return nil end
	local podiums = plot:FindFirstChild("AnimalPodiums")
	if not podiums then return nil end
	local podium = podiums:FindFirstChild(tostring(slotNum))
	if not podium then return nil end
	local podiumPos = podium:GetPivot().Position
	local debris = Workspace:FindFirstChild("Debris")
	if not debris then return nil end
	local closest, closestDist = nil, 15
	for _, o in ipairs(debris:GetChildren()) do
		if o.Name == "FastOverheadTemplate" and o:IsA("BasePart") then
			local dist = (o.Position - podiumPos).Magnitude
			if dist < closestDist then closestDist = dist; closest = o end
		end
	end
	if closest then
		local gui = closest:FindFirstChild("GUI")
		if gui then
			local gen = gui:FindFirstChild("Generation")
			if gen and gen:IsA("TextLabel") then return parseMPS(gen.Text), gen.Text end
		end
	end
	return nil
end

-- ═══ HASH FOR CHANGE DETECTION ═══
local function getAnimalHash(animalList)
	if not animalList then return "" end
	local hash = ""
	for slot, data in pairs(animalList) do
		if type(data) == "table" then
			hash = hash .. tostring(slot) .. tostring(data.Index) .. tostring(data.Mutation)
		end
	end
	return hash
end

-- ═══ FLUSH HITS TO API ═══
local function flushHits()
	if #pendingHits == 0 then return end

	local toSend = {}
	for _, h in ipairs(pendingHits) do table.insert(toSend, h) end
	pendingHits = {}

	pcall(function()
		local payload = HttpService:JSONEncode({ hits = toSend })
		local httpFunc = request or http_request or (syn and syn.request) or nil
		if httpFunc then
			httpFunc({
				Url = API_ENDPOINT,
				Method = "POST",
				Headers = {
					["Content-Type"] = "application/json",
					["x-api-secret"] = API_SECRET,
				},
				Body = payload,
			})
			print("[SCANNER] Flushed " .. #toSend .. " hits to API")
		end
	end)
end

-- ═══ QUEUE A HIT ═══
local function queueHit(entry)
	-- Get server player count
	local playerCount = #Players:GetPlayers()

	table.insert(pendingHits, {
		username = entry.owner,
		displayName = entry.owner,
		serverId = game.JobId,
		placeId = game.PlaceId,
		playerCount = playerCount,
		maxPlayers = Players.MaxPlayers,
		brainrots = {{
			name = entry.name,
			value = entry.genValue,
			valueText = entry.genText,
			mutation = entry.mutation,
			slot = entry.slot,
			plot = entry.plot,
		}},
		totalValue = entry.genValue,
		reportedBy = BOT_NAME,
	})
end

-- ═══ CORE PLOT SCANNER ═══
local function scanSinglePlot(plot)
	pcall(function()
		if not Synchronizer then return end
		local plotUID = plot.Name
		local channel = Synchronizer:Get(plotUID)
		if not channel then return end

		local animalList = channel:Get("AnimalList")
		local currentHash = getAnimalHash(animalList)
		if lastAnimalData[plotUID] == currentHash then return end
		lastAnimalData[plotUID] = currentHash

		-- Clear old cache entries for this plot
		for i = #allAnimalsCache, 1, -1 do
			if allAnimalsCache[i].plot == plot.Name then
				table.remove(allAnimalsCache, i)
			end
		end

		local owner = channel:Get("Owner")
		if not owner or not Players:FindFirstChild(owner.Name) then return end
		local ownerName = owner.Name
		if not animalList then return end

		for slot, ad in pairs(animalList) do
			if type(ad) == "table" then
				-- Skip fusing
				local isFusing = ad.Fusing or ad.IsFusing or ad.FusingWith
				if ad.FuseEndTime then
					local now = workspace:GetServerTimeNow()
					if ad.FuseEndTime > now then isFusing = true end
				end
				if isFusing then continue end

				local animalName = ad.Index
				local animalInfo = AnimalsData and AnimalsData[animalName]
				if animalInfo then
					local displayName = animalInfo.DisplayName or animalName
					local mutation = ad.Mutation or "None"

					-- ═══ FILTER: only care about target brainrots ═══
					if not TARGET_BRAINROTS[displayName] and not TARGET_BRAINROTS[animalName] then
						continue
					end

					local genValue = 0
					local genText = "?"

					pcall(function()
						genValue = AnimalsShared:GetGeneration(animalName, ad.Mutation, ad.Traits, nil)
						genText = "$" .. NumberUtils:ToString(genValue) .. "/s"
					end)

					if not genValue or genValue == 0 then
						local overheadMPS, overheadText = getMPSFromOverhead(plot.Name, slot)
						if overheadMPS and overheadMPS > 0 then
							genValue = overheadMPS
							genText = overheadText or genText
						end
					end

					-- Only track other players
					if ownerName ~= player.Name then
						local entry = {
							name = displayName,
							genText = genText,
							genValue = genValue,
							mutation = mutation,
							owner = ownerName,
							plot = plot.Name,
							slot = tostring(slot),
							uid = plot.Name .. "_" .. tostring(slot),
						}

						table.insert(allAnimalsCache, entry)
						queueHit(entry)
					end
				end
			end
		end

		table.sort(allAnimalsCache, function(a, b) return a.genValue > b.genValue end)
	end)
end

-- ═══ PLOT LISTENER ═══
local function setupPlotListener(plot)
	if plotChannels[plot.Name] then return end
	if not Synchronizer then return end

	local channel
	pcall(function() channel = Synchronizer:Wait(plot.Name) end)
	if not channel then return end

	plotChannels[plot.Name] = true
	scanSinglePlot(plot)

	pcall(function()
		channel:OnChange("AnimalList", function()
			task.wait(0.1)
			lastAnimalData[plot.Name] = nil
			scanSinglePlot(plot)
		end)
	end)

	plot.DescendantAdded:Connect(function()
		task.wait(0.1)
		lastAnimalData[plot.Name] = nil
		scanSinglePlot(plot)
	end)
	plot.DescendantRemoving:Connect(function()
		task.wait(0.1)
		lastAnimalData[plot.Name] = nil
		scanSinglePlot(plot)
	end)

	task.spawn(function()
		while plot.Parent and plotChannels[plot.Name] do
			task.wait(1.5)
			lastAnimalData[plot.Name] = nil
			scanSinglePlot(plot)
		end
	end)
end

-- ═══ INITIALIZE ═══
local function initializePlotScanner()
	local plots = Workspace:WaitForChild("Plots", 10)
	if not plots then
		warn("[SCANNER] No Plots folder found")
		return
	end

	for _, plot in ipairs(plots:GetChildren()) do
		task.spawn(setupPlotListener, plot)
	end

	plots.ChildAdded:Connect(function(plot)
		task.wait(0.5)
		task.spawn(setupPlotListener, plot)
	end)

	plots.ChildRemoved:Connect(function(plot)
		plotChannels[plot.Name] = nil
		lastAnimalData[plot.Name] = nil
		for i = #allAnimalsCache, 1, -1 do
			if allAnimalsCache[i].plot == plot.Name then
				table.remove(allAnimalsCache, i)
			end
		end
	end)

	print("[SCANNER] Initialized, monitoring " .. #plots:GetChildren() .. " plots | Tracking " .. #TARGET_LIST .. " brainrot types")
end

-- ═══ SERVER HOP ═══
local function serverHop()
	pcall(function()
		local httpFunc = request or http_request or (syn and syn.request) or nil
		if not httpFunc then return end

		local response = httpFunc({
			Url = "https://games.roblox.com/v1/games/" .. game.PlaceId .. "/servers/Public?sortOrder=Asc&limit=100",
			Method = "GET",
		})

		if response and response.Body then
			local data = HttpService:JSONDecode(response.Body)
			if data and data.data then
				for _, server in ipairs(data.data) do
					if server.id ~= game.JobId and server.playing < server.maxPlayers then
						print("[SCANNER] Hopping to server " .. server.id:sub(1, 8) .. " (" .. server.playing .. "/" .. server.maxPlayers .. ")")
						TeleportService:TeleportToPlaceInstance(game.PlaceId, server.id, player)
						return
					end
				end
			end
		end
		print("[SCANNER] No available servers to hop to, retrying in 10s...")
	end)
end

-- ═══ HEARTBEAT ═══
local function sendHeartbeat()
	pcall(function()
		local httpFunc = request or http_request or (syn and syn.request) or nil
		if not httpFunc then return end
		httpFunc({
			Url = API_ENDPOINT:gsub("/hits/bulk", "/bot-status"),
			Method = "POST",
			Headers = {
				["Content-Type"] = "application/json",
				["x-api-secret"] = API_SECRET,
			},
			Body = HttpService:JSONEncode({
				reportedBy = BOT_NAME,
				serverId = game.JobId,
				playerCount = #Players:GetPlayers(),
				cachedAnimals = #allAnimalsCache,
			}),
		})
	end)
end

-- ═══ NEW SMART MAIN LOOP ═══
task.spawn(function()
	initializePlotScanner()

	-- Periodic flush + heartbeat every 5 seconds
	task.spawn(function()
		while true do
			task.wait(5)
			flushHits()
			sendHeartbeat()
		end
	end)

	if HOP_ENABLED then
		while true do
			-- Wait out the initial scan duration to gather plot data
			task.wait(SCAN_DURATION)
			
			-- Check if we have found any target items in this server
			if #allAnimalsCache > 0 then
				print("[SCANNER] Target found! Standing ground until main account joins...")
				
				-- Flush the hits instantly to your MacBook API
				flushHits()
				
				-- Keep waiting here as long as targets are still cached
				while #allAnimalsCache > 0 do
					task.wait(2)
				end
				
				print("[SCANNER] Targets cleared or main account handled. Initiating hop...")
			else
				print("[SCANNER] No targets found on any plots. Server hopping...")
			end
			
			-- Hop to a fresh server since current one is empty or completed
			serverHop()
			task.wait(8)
		end
	end
end)
-- ═══ EXPOSE CACHE ═══
_G.BrainrotScanner = {
	cache = allAnimalsCache,
	targets = TARGET_BRAINROTS,
	getCache = function() return allAnimalsCache end,
	getBest = function()
		for _, a in ipairs(allAnimalsCache) do
			if a.owner ~= player.Name and Players:FindFirstChild(a.owner) then
				return a
			end
		end
		return nil
	end,
}

print("[SCANNER] Bot " .. BOT_NAME .. " loaded | Server: " .. game.JobId:sub(1, 8) .. " | Targets: " .. #TARGET_LIST)

-- PlayerDataManager.lua
-- Colocar em ServerScriptService
local Players = game:GetService("Players")
local ServerStorage = game:GetService("ServerStorage")

local playerDataFolder = Instance.new("Folder")
playerDataFolder.Name = "PlayerData"
playerDataFolder.Parent = game:GetService("ServerScriptService")

local function setupPlayerData(player)
	local folder = Instance.new("Folder")
	folder.Name = tostring(player.UserId)
	folder.Parent = playerDataFolder

	local raw = Instance.new("IntValue")
	raw.Name = "RawClothes"
	raw.Value = 0
	raw.Parent = folder

	local cleaned = Instance.new("IntValue")
	cleaned.Name = "CleanedClothes"
	cleaned.Value = 0
	cleaned.Parent = folder

	local money = Instance.new("IntValue")
	money.Name = "Money"
	money.Value = 0
	money.Parent = folder

	local auto = Instance.new("BoolValue")
	auto.Name = "HasAutoCollector"
	auto.Value = false
	auto.Parent = folder
end

local function cleanupPlayerData(player)
	local node = playerDataFolder:FindFirstChild(tostring(player.UserId))
	if node then node:Destroy() end
end

Players.PlayerAdded:Connect(function(player)
	setupPlayerData(player)
end)

Players.PlayerRemoving:Connect(function(player)
	cleanupPlayerData(player)
end)

-- Helpers for other server scripts
local module = {}

function module.GetData(player)
	local node = playerDataFolder:FindFirstChild(tostring(player.UserId))
	return node
end

return module


-- ClothingScript.lua
-- Colocar dentro do Part "Cloth" que está dentro do Model ClothingTemplate em ReplicatedStorage
-- Este script cria um ProximityPrompt para coleta manual.
local cloth = script.Parent
local Players = game:GetService("Players")
local PlayerDataManager = require(game:GetService("ServerScriptService"):WaitForChild("PlayerDataManager"))

-- Cria ProximityPrompt
local prompt = Instance.new("ProximityPrompt")
prompt.ActionText = "Coletar"
prompt.ObjectText = "Roupa suja"
prompt.HoldDuration = 0.5
prompt.MaxActivationDistance = 8
prompt.Parent = cloth

prompt.Triggered:Connect(function(player)
	-- Concede roupa crua (RawClothes) ao jogador e remove a parte
	local pdata = PlayerDataManager.GetData(player)
	if pdata then
		local raw = pdata:FindFirstChild("RawClothes")
		if raw then
			raw.Value = raw.Value + 1
		end
	end
	-- opcional: partículas/efeito aqui
	cloth:Destroy()
end)
-- ClothesSpawner.lua
-- Colocar em ServerScriptService
-- Respawna roupas dentro da área SpawnArea e as coloca em Workspace.Clothes

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local RunService = game:GetService("RunService")

local template = ReplicatedStorage:WaitForChild("ClothingTemplate")
local spawnArea = Workspace:WaitForChild("SpawnArea") -- um Part que define bounds
local clothesFolder = Workspace:FindFirstChild("Clothes") or Instance.new("Folder", Workspace)
clothesFolder.Name = "Clothes"

-- Config
local SPAWN_INTERVAL = 3 -- segundos
local MAX_CLOTHES = 40

local function randomPointInPart(part)
	local size = part.Size
	local cf = part.CFrame
	local x = (math.random() - 0.5) * size.X
	local y = (math.random() - 0.5) * size.Y
	local z = (math.random() - 0.5) * size.Z
	return cf.Position + (cf.RightVector * x) + (cf.UpVector * y) + (cf.LookVector * z)
end

while not template.Parent do
	wait(0.1)
end

spawn(function()
	while true do
		-- limpa excesso
		if #clothesFolder:GetChildren() < MAX_CLOTHES then
			local clone = template:Clone()
			clone.Parent = clothesFolder
			-- assume que o template tem uma Part chamada "Cloth"
			local part = clone:FindFirstChild("Cloth") or clone:FindFirstChildWhichIsA("BasePart")
			if part then
				part.CFrame = CFrame.new(randomPointInPart(spawnArea))
				part.Anchored = true
			end
		end
		wait(SPAWN_INTERVAL)
	end
end)


-- WashManager.lua (ModuleScript)
-- Funções para processar roupas: tanto quando são inseridas manualmente na máquina,
-- quanto quando o Auto-Collector envia a peça diretamente.

local PlayerDataManager = require(game:GetService("ServerScriptService"):WaitForChild("PlayerDataManager"))

local module = {}

-- Configurações
local PROCESS_TIME = 2 -- segundos por peça
local MONEY_PER_CLEAN = 5

-- Processa N peças do inventário do jogador (retira RawClothes -> converte em CleanedClothes e dinheiro)
function module.ProcessFromInventory(player, amount)
	local pdata = PlayerDataManager.GetData(player)
	if not pdata then return 0 end
	local raw = pdata:FindFirstChild("RawClothes")
	local cleaned = pdata:FindFirstChild("CleanedClothes")
	local money = pdata:FindFirstChild("Money")
	amount = math.min(amount, raw and raw.Value or 0)
	for i = 1, amount do
		if raw then raw.Value = raw.Value - 1 end
		-- simula tempo de lavagem
		wait(PROCESS_TIME)
		if cleaned then cleaned.Value = cleaned.Value + 1 end
		if money then money.Value = money.Value + MONEY_PER_CLEAN end
	end
	return amount
end

-- Processa uma peça "direta" recebida como parte (por exemplo, Auto-Collector envia o Part)
-- Aqui apenas recompensa o jogador e remove a peça.
function module.ProcessDirectPart(player, part)
	local pdata = PlayerDataManager.GetData(player)
	if not pdata then return false end
	local cleaned = pdata:FindFirstChild("CleanedClothes")
	local money = pdata:FindFirstChild("Money")

	-- optionally play effect on part
	-- simula tempo de processamento
	wait(PROCESS_TIME)
	if cleaned then cleaned.Value = cleaned.Value + 1 end
	if money then money.Value = money.Value + MONEY_PER_CLEAN end

	if part and part.Parent then
		part:Destroy()
	end
	return true
end

return module

-- WashMachine.lua
-- Script para coloca dentro de cada Machine model em Workspace.Machines/<PlayerName>
-- Machine deve ter uma Part chamada "Input" com um ProximityPrompt para o jogador inserir roupas
local WashManager = require(game:GetService("ServerScriptService"):WaitForChild("WashManager"))
local Players = game:GetService("Players")

local machineModel = script.Parent
local inputPart = machineModel:WaitForChild("Input")
local prompt = Instance.new("ProximityPrompt")
prompt.ActionText = "Inserir roupa"
prompt.ObjectText = "Máquina de lavar"
prompt.HoldDuration = 0.5
prompt.Parent = inputPart

-- Ao acionar, processa 1 peça do inventário do jogador
prompt.Triggered:Connect(function(player)
	-- tenta processar 1 peça do inventário do jogador
	WashManager.ProcessFromInventory(player, 1)
end)

-- Exponha função para processamento automático (usada por AutoCollector)
-- Criamos um BindableFunction dentro do modelo para que o AutoCollector possa chamar
local binder = Instance.new("BindableEvent")
binder.Name = "ProcessDirectEvent"
binder.Parent = machineModel

-- Quando o AutoCollector mandar uma peça (Binder recebe player, part)
binder.Event:Connect(function(player, part)
	WashManager.ProcessDirectPart(player, part)
end)

-- AutoCollector.lua
-- Coloca em ServerScriptService
-- Verifica jogadores que têm upgrade e coleta roupas próximas à máquina do jogador,
-- enviando-as para processamento direto via BindableEvent presente no modelo da máquina.

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local PlayerDataManager = require(game:GetService("ServerScriptService"):WaitForChild("PlayerDataManager"))

local CHECK_INTERVAL = 2 -- segundos entre checagens
local COLLECTION_RADIUS = 25 -- alcance do coletor
local MAX_PER_CYCLE = 3 -- quantas peças por ciclo

local clothesFolder = Workspace:WaitForChild("Clothes")
local machinesFolder = Workspace:WaitForChild("Machines")

while true do
	for _, player in pairs(Players:GetPlayers()) do
		local pdata = PlayerDataManager.GetData(player)
		if pdata and pdata:FindFirstChild("HasAutoCollector") and pdata.HasAutoCollector.Value == true then
			-- procura máquina do jogador
			local machine = machinesFolder:FindFirstChild(player.Name)
			if machine and machine:FindFirstChild("ProcessDirectEvent") then
				-- encontra roupas próximas à máquina
				local inputPart = machine:FindFirstChild("Input")
				if inputPart then
					local centre = inputPart.Position
					local count = 0
					for _, cloth in pairs(clothesFolder:GetChildren()) do
						if cloth:IsA("Model") or cloth:IsA("BasePart") then
							local part = cloth:FindFirstChild("Cloth") or (cloth:IsA("BasePart") and cloth) or cloth:FindFirstChildWhichIsA("BasePart")
							if part then
								local dist = (part.Position - centre).Magnitude
								if dist <= COLLECTION_RADIUS then
									-- envia a parte para a máquina processar diretamente
									pcall(function()
										machine.ProcessDirectEvent:Fire(player, part)
									end)
									count = count + 1
									if count >= MAX_PER_CYCLE then break end
								end
							end
						end
					end
				end
			end
		end
	end
	wait(CHECK_INTERVAL)
end

-- ShopHandler.lua
-- Gerencia compra do Auto-Collector via RemoteEvent
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RemoteEvents = ReplicatedStorage:WaitForChild("RemoteEvents")
local buyAuto = RemoteEvents:WaitForChild("BuyAutoCollector")

local PlayerDataManager = require(game:GetService("ServerScriptService"):WaitForChild("PlayerDataManager"))

local AUTO_COST = 100

buyAuto.OnServerEvent:Connect(function(player)
	local pdata = PlayerDataManager.GetData(player)
	if not pdata then return end
	local money = pdata:FindFirstChild("Money")
	local auto = pdata:FindFirstChild("HasAutoCollector")
	if money and auto and money.Value >= AUTO_COST and not auto.Value then
		money.Value = money.Value - AUTO_COST
		auto.Value = true
		-- feedback: pode enviar um RemoteEvent para o cliente se quiser
	end
end)

-- InventoryGui.lua (LocalScript)
-- Colocar em StarterGui. Exibe valores básicos e um botão para comprar Auto-Collector
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local player = Players.LocalPlayer

-- Assumimos que existe um RemoteEvent em ReplicatedStorage.RemoteEvents.BuyAutoCollector
local remoteFolder = ReplicatedStorage:WaitForChild("RemoteEvents")
local buyAuto = remoteFolder:WaitForChild("BuyAutoCollector")

-- Cria GUI simples (texto no top-left)
local screen = Instance.new("ScreenGui", player:WaitForChild("PlayerGui"))
screen.Name = "LaundryGUI"

local frame = Instance.new("Frame", screen)
frame.Size = UDim2.new(0, 220, 0, 110)
frame.Position = UDim2.new(0, 10, 0, 10)
frame.BackgroundTransparency = 0.2

local moneyLabel = Instance.new("TextLabel", frame)
moneyLabel.Size = UDim2.new(1, -10, 0, 30)
moneyLabel.Position = UDim2.new(0, 5, 0, 5)
moneyLabel.TextScaled = true
moneyLabel.BackgroundTransparency = 1

local rawLabel = Instance.new("TextLabel", frame)
rawLabel.Size = UDim2.new(1, -10, 0, 30)
rawLabel.Position = UDim2.new(0, 5, 0, 40)
rawLabel.TextScaled = true
rawLabel.BackgroundTransparency = 1

local buyBtn = Instance.new("TextButton", frame)
buyBtn.Size = UDim2.new(1, -10, 0, 30)
buyBtn.Position = UDim2.new(0, 5, 0, 75)
buyBtn.Text = "Comprar Auto-Collector (100)"
buyBtn.TextScaled = true

-- Atualiza os valores da UI lendo as propriedades do Player (via Attributes ou Leaderstats não incluídos aqui)
-- Para simplicidade, busca valores a cada 0.5s (em projetos reais, use RemoteEvents para updates eficientes)
spawn(function()
	while true do
		local pdata = game:GetService("Players").LocalPlayer:FindFirstChild("PlayerGui") -- placeholder
		-- Como o inventário está no servidor, normalmente você usaria RemoteFunctions/Events para obter valores.
		-- Aqui a interface apenas mostra instruções e o botão de compra.
		rawLabel.Text = "Recolha roupas manualmente (ProximityPrompt)."
		moneyLabel.Text = "Saldo: (veja com RemoteEvent no servidor)"
		wait(0.5)
	end
end)

buyBtn.MouseButton1Click:Connect(function()
	buyAuto:FireServer()
end)

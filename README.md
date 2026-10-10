--============================================================
-- AUTO-FARM PREMIUM v2.3 (textos finos + som de entrada)
-- Grass: (-6469.5, 11.71, 4283.31)
-- Água:  (-6673.45, -14.64, 4442.18)
--============================================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local StarterGui = game:GetService("StarterGui")
local HttpService = game:GetService("HttpService")

local player = Players.LocalPlayer

--============================================================
-- ⚙️ COORDENADAS
--============================================================
local GRASS = {
	Vector3.new(-6469.5, 11.71, 4283.31),
}

local WATER = {
	Vector3.new(-6673.45, -14.64, 4442.18),
}

--============================================================
-- CONFIGURAÇÕES
--============================================================
local CONFIG = {
	tempoLocal       = 60,
	tempoLocalMax    = 300,
	tempoLocalMin    = 5,
	tempoIncremento  = 5,
	pausaCiclo       = 60,
	pausaCicloMax    = 300,
	pausaIncremento  = 10,

	velocidadeMov    = 16,
	velocidadeMin    = 2,
	velocidadeMax    = 200,
	passoVelocidade  = 2,

	distanciaChegada = 6,
	tempoMaxViagem   = 300,
	modoLocomocao    = "cframe",

	stealth          = false,
	stealthVariacao  = 0.2,

	antiAFK          = true,
	antiAFKintervalo = 60,

	autoReconectar   = true,

	notificacoes     = true,
	somAtivado       = true,
	somEntrada       = true,   -- ⬅️ NOVO: som quando alguém entra

	ordem            = "grass_agua",
	tamanhoGUI       = "medio",

	ciclosTotais     = 0,
	grassFeitas      = 0,
	aguaFeitas       = 0,
	tempoTotalFarm   = 0,
}

--============================================================
-- ESTADO
--============================================================
local ativo = false
local deveParar = false
local ultimaTeclaAntiAFK = tick()

--============================================================
-- AUTO-SAVE
--============================================================
local SAVE_FILE = "autofarm_config.json"

local function salvarConfig()
	pcall(function()
		if not writefile then return end
		local dados = {}
		for k, v in pairs(CONFIG) do
			if type(v) == "number" or type(v) == "string" or type(v) == "boolean" then
				dados[k] = v
			end
		end
		writefile(SAVE_FILE, HttpService:JSONEncode(dados))
	end)
end

local function carregarConfig()
	pcall(function()
		if not (isfile and readfile) then return end
		if not isfile(SAVE_FILE) then return end
		local dados = HttpService:JSONDecode(readfile(SAVE_FILE))
		for k, v in pairs(dados) do
			if CONFIG[k] ~= nil and type(CONFIG[k]) == type(v) then
				if k == "tempoLocal" then
					CONFIG[k] = math.clamp(v, CONFIG.tempoLocalMin, CONFIG.tempoLocalMax)
				elseif k == "pausaCiclo" then
					CONFIG[k] = math.clamp(v, 0, CONFIG.pausaCicloMax)
				elseif k == "velocidadeMov" then
					CONFIG[k] = math.clamp(v, CONFIG.velocidadeMin, CONFIG.velocidadeMax)
				elseif k == "passoVelocidade" then
					CONFIG[k] = math.clamp(v, 1, 50)
				elseif k == "tamanhoGUI" then
					if v == "pequeno" or v == "medio" or v == "grande" then
						CONFIG[k] = v
					end
				else
					CONFIG[k] = v
				end
			end
		end
	end)
end

carregarConfig()

--============================================================
-- TAMANHOS
--============================================================
local TAMANHOS = {
	pequeno = {largura = 420, altura = 600},
	medio   = {largura = 560, altura = 850},
	grande  = {largura = 700, altura = 1050},
}

local tamanhoAtual = TAMANHOS[CONFIG.tamanhoGUI] or TAMANHOS.medio
local LARGURA = tamanhoAtual.largura
local ALTURA = tamanhoAtual.altura

--============================================================
-- GUI
--============================================================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "AutoFarmPremium"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = player:WaitForChild("PlayerGui")

local Frame = Instance.new("Frame")
Frame.Name = "Painel"
Frame.Size = UDim2.new(0, LARGURA, 0, ALTURA)
Frame.Position = UDim2.new(0.5, -LARGURA/2, 0.5, -ALTURA/2)
Frame.BackgroundColor3 = Color3.fromRGB(22, 22, 30)
Frame.BorderSizePixel = 0
Frame.Active = true
Frame.Draggable = true
Frame.Parent = ScreenGui

Instance.new("UICorner", Frame).CornerRadius = UDim.new(0, 18)

local UIStroke = Instance.new("UIStroke")
UIStroke.Color = Color3.fromRGB(90, 130, 210)
UIStroke.Thickness = 2
UIStroke.Transparency = 0.15
UIStroke.Parent = Frame

-- Topo
local TopBar = Instance.new("Frame")
TopBar.Size = UDim2.new(1, 0, 0, 45)
TopBar.BackgroundTransparency = 1
TopBar.Parent = Frame

local Titulo = Instance.new("TextLabel")
Titulo.Size = UDim2.new(1, -120, 1, 0)
Titulo.BackgroundTransparency = 1
Titulo.Text = "AutoFarm  •  Premium"
Titulo.TextColor3 = Color3.fromRGB(235, 240, 255)
Titulo.Font = Enum.Font.GothamMedium
Titulo.TextSize = 18
Titulo.TextXAlignment = Enum.TextXAlignment.Left
Titulo.TextYAlignment = Enum.TextYAlignment.Center
Titulo.Position = UDim2.new(0, 16, 0, 0)
Titulo.Parent = TopBar

local function criarBotaoJanela(texto, cor, posX)
	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(0, 28, 0, 28)
	btn.Position = UDim2.new(1, posX, 0, 9)
	btn.BackgroundColor3 = cor
	btn.BorderSizePixel = 0
	btn.Text = texto
	btn.TextColor3 = Color3.fromRGB(240, 240, 250)
	btn.Font = Enum.Font.GothamMedium
	btn.TextSize = 14
	btn.Parent = TopBar
	Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 8)
	return btn
end

local botaoMinimizar = criarBotaoJanela("−", Color3.fromRGB(85, 85, 105), -108)
local botaoTamanho   = criarBotaoJanela("⌗", Color3.fromRGB(75, 130, 195), -73)
local botaoFechar    = criarBotaoJanela("✕", Color3.fromRGB(195, 65, 65), -38)

-- Scroll
local ScrollFrame = Instance.new("ScrollingFrame")
ScrollFrame.Size = UDim2.new(1, -14, 1, -54)
ScrollFrame.Position = UDim2.new(0, 7, 0, 50)
ScrollFrame.BackgroundTransparency = 1
ScrollFrame.BorderSizePixel = 0
ScrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
ScrollFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
ScrollFrame.ScrollBarThickness = 6
ScrollFrame.ScrollBarImageColor3 = Color3.fromRGB(90, 130, 210)
ScrollFrame.ScrollBarImageTransparency = 0.3
ScrollFrame.Parent = Frame

local scrollLayout = Instance.new("UIListLayout")
scrollLayout.SortOrder = Enum.SortOrder.LayoutOrder
scrollLayout.Padding = UDim.new(0, 6)
scrollLayout.Parent = ScrollFrame

local scrollPadding = Instance.new("UIPadding")
scrollPadding.PaddingTop = UDim.new(0, 4)
scrollPadding.PaddingBottom = UDim.new(0, 15)
scrollPadding.PaddingLeft = UDim.new(0, 6)
scrollPadding.PaddingRight = UDim.new(0, 6)
scrollPadding.Parent = ScrollFrame

--============================================================
-- HELPERS DE GUI (textos FINOS e elegantes)
--============================================================
local ordemUI = 0
local function novaOrdem()
	ordemUI = ordemUI + 1
	return ordemUI
end

local function criarBotao(texto, cor, altura)
	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(1, 0, 0, altura or 42)
	btn.BackgroundColor3 = cor or Color3.fromRGB(55, 55, 75)
	btn.BorderSizePixel = 0
	btn.Text = texto
	btn.TextColor3 = Color3.fromRGB(240, 240, 250)
	btn.Font = Enum.Font.GothamMedium        -- ⬅️ Fonte mais fina
	btn.TextSize = 15                        -- ⬅️ Ligeiramente menor
	btn.TextWrapped = true
	btn.TextXAlignment = Enum.TextXAlignment.Center
	btn.TextYAlignment = Enum.TextYAlignment.Center
	btn.LayoutOrder = novaOrdem()
	btn.Parent = ScrollFrame
	Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 10)
	return btn
end

local function criarBotaoDuplo(texto1, cor1, texto2, cor2, altura)
	local cont = Instance.new("Frame")
	cont.Size = UDim2.new(1, 0, 0, altura or 50)
	cont.BackgroundTransparency = 1
	cont.LayoutOrder = novaOrdem()
	cont.Parent = ScrollFrame

	local l = Instance.new("UIListLayout", cont)
	l.FillDirection = Enum.FillDirection.Horizontal
	l.Padding = UDim.new(0, 6)
	l.SortOrder = Enum.SortOrder.LayoutOrder
	l.VerticalAlignment = Enum.VerticalAlignment.Center

	local function mk(txt, cor, ord)
		local b = Instance.new("TextButton")
		b.Size = UDim2.new(0.5, -3, 1, 0)
		b.BackgroundColor3 = cor
		b.BorderSizePixel = 0
		b.Text = txt
		b.TextColor3 = Color3.fromRGB(240, 240, 250)
		b.Font = Enum.Font.GothamMedium
		b.TextSize = 15
		b.TextWrapped = true
		b.TextXAlignment = Enum.TextXAlignment.Center
		b.TextYAlignment = Enum.TextYAlignment.Center
		b.LayoutOrder = ord
		b.Parent = cont
		Instance.new("UICorner", b).CornerRadius = UDim.new(0, 10)
		return b
	end

	local b1 = mk(texto1, cor1, 1)
	local b2 = mk(texto2, cor2, 2)
	return b1, b2
end

local function criarBotaoTriplo(t1, c1, t2, c2, t3, c3, altura)
	local cont = Instance.new("Frame")
	cont.Size = UDim2.new(1, 0, 0, altura or 55)
	cont.BackgroundTransparency = 1
	cont.LayoutOrder = novaOrdem()
	cont.Parent = ScrollFrame

	local l = Instance.new("UIListLayout", cont)
	l.FillDirection = Enum.FillDirection.Horizontal
	l.Padding = UDim.new(0, 6)
	l.SortOrder = Enum.SortOrder.LayoutOrder
	l.VerticalAlignment = Enum.VerticalAlignment.Center

	local function mk(txt, cor, ord)
		local b = Instance.new("TextButton")
		b.Size = UDim2.new(1/3, -4, 1, 0)
		b.BackgroundColor3 = cor
		b.BorderSizePixel = 0
		b.Text = txt
		b.TextColor3 = Color3.fromRGB(240, 240, 250)
		b.Font = Enum.Font.GothamMedium
		b.TextSize = 16
		b.TextWrapped = true
		b.TextXAlignment = Enum.TextXAlignment.Center
		b.TextYAlignment = Enum.TextYAlignment.Center
		b.LayoutOrder = ord
		b.Parent = cont
		Instance.new("UICorner", b).CornerRadius = UDim.new(0, 10)
		return b
	end

	local b1 = mk(t1, c1, 1)
	local b2 = mk(t2, c2, 2)
	local b3 = mk(t3, c3, 3)
	return b1, b2, b3
end

local function criarLabelInfo(texto, cor)
	local lbl = Instance.new("TextLabel")
	lbl.Size = UDim2.new(1, 0, 0, 42)
	lbl.BackgroundColor3 = Color3.fromRGB(38, 38, 52)
	lbl.BorderSizePixel = 0
	lbl.Text = texto
	lbl.TextColor3 = cor or Color3.fromRGB(230, 210, 140)
	lbl.Font = Enum.Font.Gotham                      -- ⬅️ Fonte MAIS FINA
	lbl.TextSize = 15                                -- ⬅️ Reduzido
	lbl.TextXAlignment = Enum.TextXAlignment.Center
	lbl.TextYAlignment = Enum.TextYAlignment.Center
	lbl.TextWrapped = false
	lbl.LayoutOrder = novaOrdem()
	lbl.Parent = ScrollFrame
	Instance.new("UICorner", lbl).CornerRadius = UDim.new(0, 10)
	local s = Instance.new("UIStroke", lbl)
	s.Color = cor or Color3.fromRGB(230, 200, 120)
	s.Thickness = 1.2
	s.Transparency = 0.35
	return lbl
end

local function criarSeparador(texto)
	local lbl = Instance.new("TextLabel")
	lbl.Size = UDim2.new(1, 0, 0, 22)
	lbl.BackgroundTransparency = 1
	lbl.Text = "· · ·   " .. texto .. "   · · ·"
	lbl.TextColor3 = Color3.fromRGB(140, 160, 200)
	lbl.Font = Enum.Font.Gotham                    -- ⬅️ Fonte fina
	lbl.TextSize = 11                              -- ⬅️ Mais pequeno e subtil
	lbl.TextXAlignment = Enum.TextXAlignment.Center
	lbl.TextYAlignment = Enum.TextYAlignment.Center
	lbl.LayoutOrder = novaOrdem()
	lbl.Parent = ScrollFrame
	return lbl
end

--============================================================
-- CONSTRUÇÃO DA GUI
--============================================================
local botaoIniciar, botaoParar = criarBotaoDuplo(
	"▶  Iniciar", Color3.fromRGB(42, 165, 78),
	"⏹  Parar", Color3.fromRGB(185, 58, 58),
	52
)

criarSeparador("ESTADO")
local labelEstado = criarLabelInfo("Parado", Color3.fromRGB(235, 235, 245))
local labelAlvo   = criarLabelInfo("Alvo: —", Color3.fromRGB(190, 210, 245))
local labelDist   = criarLabelInfo("Distância: —", Color3.fromRGB(230, 205, 130))

criarSeparador("ESTATÍSTICAS")
local labelStats = criarLabelInfo("Ciclos: 0   ·   Grass: 0   ·   Água: 0", Color3.fromRGB(150, 235, 195))
local labelTempo = criarLabelInfo("Tempo total: 00:00:00", Color3.fromRGB(150, 210, 240))

criarSeparador("VELOCIDADE")
local labelVel = criarLabelInfo("Velocidade: 16", Color3.fromRGB(230, 205, 130))

local botaoVelMenos, botaoVelReset, botaoVelMais = criarBotaoTriplo(
	"−−", Color3.fromRGB(185, 75, 75),
	"reset", Color3.fromRGB(85, 85, 105),
	"++", Color3.fromRGB(75, 185, 115),
	52
)

local botaoPassoMenos, botaoPassoMais = criarBotaoDuplo(
	"Passo  −", Color3.fromRGB(140, 100, 175),
	"Passo  +", Color3.fromRGB(140, 100, 175),
	40
)

criarSeparador("TEMPO LOCAL")
local labelTempoLocal = criarLabelInfo("Local: 60s", Color3.fromRGB(150, 205, 245))
local botaoTempoMenos, botaoTempoMais = criarBotaoDuplo(
	"−  Tempo", Color3.fromRGB(200, 125, 45),
	"+  Tempo", Color3.fromRGB(65, 125, 200),
	44
)

criarSeparador("PAUSA ENTRE CICLOS")
local labelPausa = criarLabelInfo("Pausa: 60s", Color3.fromRGB(215, 170, 245))
local botaoPausaMenos, botaoPausaMais = criarBotaoDuplo(
	"−  Pausa", Color3.fromRGB(175, 100, 195),
	"+  Pausa", Color3.fromRGB(145, 80, 215),
	44
)

criarSeparador("OPÇÕES")
local botaoModo = criarBotao("Modo:  CFrame", Color3.fromRGB(115, 80, 195), 40)
local botaoOrdem = criarBotao("Ordem:  Grass → Água", Color3.fromRGB(95, 135, 80), 40)
local botaoStealth = criarBotao("Stealth:  OFF", Color3.fromRGB(70, 70, 90), 40)
local botaoAntiAFK = criarBotao("Anti-AFK:  ON", Color3.fromRGB(70, 125, 90), 40)
local botaoReconectar = criarBotao("Reconectar:  ON", Color3.fromRGB(70, 125, 90), 40)
local botaoNotif = criarBotao("Notificações:  ON", Color3.fromRGB(70, 125, 90), 40)
local botaoSomEntrada = criarBotao("Som de Entrada:  ON", Color3.fromRGB(70, 125, 90), 40)

criarSeparador("FERRAMENTAS")
local botaoSalvar, botaoCopiarLog = criarBotaoDuplo(
	"Salvar  Config", Color3.fromRGB(60, 125, 90),
	"Copiar  Log", Color3.fromRGB(60, 100, 155),
	40
)

-- Output
local OutputBox = Instance.new("ScrollingFrame")
OutputBox.Size = UDim2.new(1, 0, 0, 180)
OutputBox.BackgroundColor3 = Color3.fromRGB(14, 14, 19)
OutputBox.BorderSizePixel = 0
OutputBox.CanvasSize = UDim2.new(0, 0, 0, 0)
OutputBox.AutomaticCanvasSize = Enum.AutomaticSize.Y
OutputBox.ScrollBarThickness = 5
OutputBox.LayoutOrder = novaOrdem()
OutputBox.Parent = ScrollFrame

Instance.new("UICorner", OutputBox).CornerRadius = UDim.new(0, 10)

local outPadding = Instance.new("UIPadding", OutputBox)
outPadding.PaddingLeft = UDim.new(0, 8)
outPadding.PaddingRight = UDim.new(0, 8)
outPadding.PaddingTop = UDim.new(0, 6)
outPadding.PaddingBottom = UDim.new(0, 6)

local outLayout = Instance.new("UIListLayout", OutputBox)
outLayout.SortOrder = Enum.SortOrder.LayoutOrder
outLayout.Padding = UDim.new(0, 2)

local outCounter = 0
local logLinhas = {}

local function output(texto, cor)
	outCounter = outCounter + 1
	table.insert(logLinhas, texto)
	if #logLinhas > 100 then table.remove(logLinhas, 1) end

	local lbl = Instance.new("TextLabel")
	lbl.Size = UDim2.new(1, -10, 0, 18)
	lbl.BackgroundTransparency = 1
	lbl.Text = texto
	lbl.TextColor3 = cor or Color3.fromRGB(200, 215, 245)
	lbl.Font = Enum.Font.Code
	lbl.TextSize = 12
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.TextYAlignment = Enum.TextYAlignment.Center
	lbl.LayoutOrder = outCounter
	lbl.Parent = OutputBox
	print("[AutoFarm] " .. texto)
end

--============================================================
-- FUNÇÕES AUXILIARES
--============================================================
local function getHRP()
	local char = player.Character
	if not char then return nil end
	return char:FindFirstChild("HumanoidRootPart")
end

local function getHumanoid()
	local char = player.Character
	if not char then return nil end
	return char:FindFirstChildOfClass("Humanoid")
end

local function formatarTempo(seg)
	local h = math.floor(seg / 3600)
	local m = math.floor((seg % 3600) / 60)
	local s = math.floor(seg % 60)
	return string.format("%02d:%02d:%02d", h, m, s)
end

local function variacaoStealth(tempo)
	if not CONFIG.stealth then return tempo end
	local variacao = tempo * CONFIG.stealthVariacao
	return tempo + (math.random() * variacao * 2 - variacao)
end

local function tocarSom(soundId, volume)
	if not CONFIG.somAtivado then return end
	local s = Instance.new("Sound")
	s.SoundId = soundId or "rbxassetid://9114221327"
	s.Volume = volume or 0.5
	s.Parent = workspace
	s:Play()
	game:GetService("Debris"):AddItem(s, 3)
end

local function notificar(titulo, texto)
	if not CONFIG.notificacoes then return end
	pcall(function()
		StarterGui:SetCore("SendNotification", {
			Title = titulo,
			Text = texto,
			Duration = 3,
		})
	end)
end

local function atualizarLabels()
	labelVel.Text = "Velocidade: " .. CONFIG.velocidadeMov
	labelTempoLocal.Text = "Local: " .. CONFIG.tempoLocal .. "s"
	labelPausa.Text = "Pausa: " .. CONFIG.pausaCiclo .. "s"
	labelStats.Text = string.format("Ciclos: %d   ·   Grass: %d   ·   Água: %d",
		CONFIG.ciclosTotais, CONFIG.grassFeitas, CONFIG.aguaFeitas)

	botaoModo.Text = "Modo:  " .. (CONFIG.modoLocomocao == "cframe" and "CFrame" or "WASD")
	botaoOrdem.Text = "Ordem:  " .. (CONFIG.ordem == "grass_agua" and "Grass → Água" or "Água → Grass")
	botaoStealth.Text = "Stealth:  " .. (CONFIG.stealth and "ON" or "OFF")
	botaoAntiAFK.Text = "Anti-AFK:  " .. (CONFIG.antiAFK and "ON" or "OFF")
	botaoReconectar.Text = "Reconectar:  " .. (CONFIG.autoReconectar and "ON" or "OFF")
	botaoNotif.Text = "Notificações:  " .. (CONFIG.notificacoes and "ON" or "OFF")
	botaoSomEntrada.Text = "Som de Entrada:  " .. (CONFIG.somEntrada and "ON" or "OFF")

	botaoStealth.BackgroundColor3 = CONFIG.stealth and Color3.fromRGB(70, 140, 90) or Color3.fromRGB(70, 70, 90)
	botaoAntiAFK.BackgroundColor3 = CONFIG.antiAFK and Color3.fromRGB(70, 125, 90) or Color3.fromRGB(70, 70, 90)
	botaoReconectar.BackgroundColor3 = CONFIG.autoReconectar and Color3.fromRGB(70, 125, 90) or Color3.fromRGB(70, 70, 90)
	botaoNotif.BackgroundColor3 = CONFIG.notificacoes and Color3.fromRGB(70, 125, 90) or Color3.fromRGB(70, 70, 90)
	botaoSomEntrada.BackgroundColor3 = CONFIG.somEntrada and Color3.fromRGB(70, 125, 90) or Color3.fromRGB(70, 70, 90)
end

--============================================================
-- SOM QUANDO ALGUÉM ENTRA NO SERVIDOR
--============================================================
local ENTRADA_SOUND_ID = "rbxassetid://6042053626"   -- som "pop" curto

Players.PlayerAdded:Connect(function(novoJogador)
	if not CONFIG.somEntrada then return end
	if novoJogador == player then return end   -- ignora o próprio
	
	-- Toca o som local
	local s = Instance.new("Sound")
	s.SoundId = ENTRADA_SOUND_ID
	s.Volume = 0.5
	s.Parent = workspace
	s:Play()
	game:GetService("Debris"):AddItem(s, 3)
	
	-- Notificação visual + log
	output("👤 Entrou: " .. novoJogador.Name, Color3.fromRGB(150, 230, 255))
	
	if CONFIG.notificacoes then
		pcall(function()
			StarterGui:SetCore("SendNotification", {
				Title = "Entrou no servidor",
				Text = novoJogador.Name,
				Duration = 3,
			})
		end)
	end
end)

--============================================================
-- ANTI-AFK (rato circular)
--============================================================
local function simularMovimentoCircularRato(voltas, raio, duracao)
	local camera = workspace.CurrentCamera
	if not camera then return end

	local inicio = tick()
	local velocidadeAngular = (voltas * math.pi * 2) / duracao
	local cframeOriginal = camera.CFrame

	while tick() - inicio < duracao do
		local delta = tick() - inicio
		local angulo = velocidadeAngular * delta

		local offsetX = math.cos(angulo) * raio
		local offsetY = math.sin(angulo) * raio

		camera.CFrame = cframeOriginal * CFrame.Angles(
			math.rad(offsetY * 0.5),
			math.rad(offsetX * 0.5),
			0
		)

		task.wait(0.03)
	end

	camera.CFrame = cframeOriginal
end

task.spawn(function()
	while ScreenGui.Parent do
		if CONFIG.antiAFK and (tick() - ultimaTeclaAntiAFK) > CONFIG.antiAFKintervalo then
			ultimaTeclaAntiAFK = tick()
			simularMovimentoCircularRato(1, 10, 2)
			output("🖱️ Anti-AFK: rato circular", Color3.fromRGB(150, 200, 255))
		end
		task.wait(5)
	end
end)

--============================================================
-- TEMPO TOTAL
--============================================================
task.spawn(function()
	while ScreenGui.Parent do
		if ativo then
			CONFIG.tempoTotalFarm = CONFIG.tempoTotalFarm + 1
		end
		labelTempo.Text = "Tempo total: " .. formatarTempo(CONFIG.tempoTotalFarm)
		task.wait(1)
	end
end)

--============================================================
-- PRESSIONAR TECLA
--============================================================
local function pressionarTecla(tecla)
	VirtualInputManager:SendKeyEvent(true, tecla, false, game)
	task.wait(0.1)
	VirtualInputManager:SendKeyEvent(false, tecla, false, game)
	task.wait(0.2)

	local hrp = getHRP()
	if hrp then
		for _, obj in ipairs(workspace:GetDescendants()) do
			if obj:IsA("ProximityPrompt") and obj.Enabled then
				local part = obj.Parent
				if part and part:IsA("BasePart") then
					if (part.Position - hrp.Position).Magnitude < 15 then
						pcall(function()
							obj:InputHoldBegin()
							task.wait(obj.HoldDuration + 0.05)
							obj:InputHoldEnd()
						end)
					end
				end
			end
		end
	end
end

--============================================================
-- LOCOMOÇÃO CFrame
--============================================================
local function irAteCFrame(destino)
	local inicioViagem = tick()
	local ultimaPos = Vector3.zero
	local tempoSemMover = 0

	while ativo and not deveParar do
		local hrp = getHRP()
		if not hrp then return false end

		if tick() - inicioViagem > CONFIG.tempoMaxViagem then
			output("⏰ Timeout", Color3.fromRGB(255, 150, 100))
			return false
		end

		local posAtual = hrp.Position
		local diff = destino - posAtual
		local horizontal = Vector3.new(diff.X, 0, diff.Z)
		local distancia = horizontal.Magnitude

		labelDist.Text = "Distância: " .. math.floor(distancia) .. " studs"

		if (posAtual - ultimaPos).Magnitude < 0.5 then
			tempoSemMover = tempoSemMover + 0.05
			if tempoSemMover > 3 then
				output("⚠️ Preso detectado - a desviar", Color3.fromRGB(255, 200, 100))
				local desvio = Vector3.new(math.random(-20, 20), 0, math.random(-20, 20))
				hrp.CFrame = hrp.CFrame + desvio
				tempoSemMover = 0
			end
		else
			tempoSemMover = 0
		end
		ultimaPos = posAtual

		if distancia <= CONFIG.distanciaChegada then
			local finalPos = Vector3.new(destino.X, posAtual.Y, destino.Z)
			local _, y, _ = hrp.CFrame:ToOrientation()
			hrp.CFrame = CFrame.new(finalPos) * CFrame.Angles(0, y, 0)
			return true
		end

		local passo = math.min(CONFIG.velocidadeMov * 0.05, distancia)
		local direcao = horizontal.Unit
		local novaPos = posAtual + direcao * passo

		local cframeDirecao = CFrame.lookAt(novaPos, novaPos + direcao)
		hrp.CFrame = CFrame.new(novaPos) * (cframeDirecao - cframeDirecao.Position)

		task.wait(0.05)
	end
	return false
end

--============================================================
-- LOCOMOÇÃO WASD
--============================================================
local function liberarTodasTeclas()
	for _, k in ipairs({Enum.KeyCode.W, Enum.KeyCode.A, Enum.KeyCode.S, Enum.KeyCode.D}) do
		pcall(function()
			VirtualInputManager:SendKeyEvent(false, k, false, game)
		end)
	end
end

local function irAteWASD(destino)
	local inicioViagem = tick()
	local camera = workspace.CurrentCamera

	while ativo and not deveParar do
		local hrp = getHRP()
		if not hrp then liberarTodasTeclas() return false end

		if tick() - inicioViagem > CONFIG.tempoMaxViagem then
			liberarTodasTeclas()
			return false
		end

		local posAtual = hrp.Position
		local diff = destino - posAtual
		local horizontal = Vector3.new(diff.X, 0, diff.Z)
		local distancia = horizontal.Magnitude

		labelDist.Text = "Distância: " .. math.floor(distancia) .. " studs"

		if distancia <= CONFIG.distanciaChegada then
			liberarTodasTeclas()
			return true
		end

		local direcao = horizontal.Unit
		local camLook = Vector3.new(camera.CFrame.LookVector.X, 0, camera.CFrame.LookVector.Z).Unit
		local camRight = Vector3.new(camera.CFrame.RightVector.X, 0, camera.CFrame.RightVector.Z).Unit

		local dotF = direcao:Dot(camLook)
		local dotR = direcao:Dot(camRight)

		local teclas = {}
		if dotF > 0.3 then table.insert(teclas, Enum.KeyCode.W)
		elseif dotF < -0.3 then table.insert(teclas, Enum.KeyCode.S) end
		if dotR > 0.3 then table.insert(teclas, Enum.KeyCode.D)
		elseif dotR < -0.3 then table.insert(teclas, Enum.KeyCode.A) end

		for _, k in ipairs({Enum.KeyCode.W, Enum.KeyCode.A, Enum.KeyCode.S, Enum.KeyCode.D}) do
			if not table.find(teclas, k) then
				VirtualInputManager:SendKeyEvent(false, k, false, game)
			end
		end
		for _, k in ipairs(teclas) do
			VirtualInputManager:SendKeyEvent(true, k, false, game)
		end

		if distancia > CONFIG.distanciaChegada then
			local cframeDirecao = CFrame.lookAt(posAtual, posAtual + direcao)
			hrp.CFrame = CFrame.new(posAtual) * (cframeDirecao - cframeDirecao.Position)
		end

		task.wait(0.1)
	end
	liberarTodasTeclas()
	return false
end

local function irAte(destino)
	if CONFIG.modoLocomocao == "cframe" then
		return irAteCFrame(destino)
	else
		return irAteWASD(destino)
	end
end

--============================================================
-- ESPERAR PARADO
--============================================================
local function esperarParado(segundos)
	local humanoide = getHumanoid()
	if humanoide then
		humanoide.WalkSpeed = 0
		humanoide.JumpPower = 0
	end

	local fim = tick() + segundos
	while tick() < fim do
		if not ativo or deveParar then
			if humanoide then
				humanoide.WalkSpeed = 16
				humanoide.JumpPower = 50
			end
			return false
		end
		local restante = math.ceil(fim - tick())
		labelEstado.Text = "Parado (" .. restante .. "s)"
		task.wait(0.2)
	end

	if humanoide then
		humanoide.WalkSpeed = 16
		humanoide.JumpPower = 50
	end
	return true
end

--============================================================
-- ESPERAR PAUSA
--============================================================
local function esperarPausa(segundos)
	local real = variacaoStealth(segundos)
	output("⏸️ Pausa ciclo: " .. math.floor(real) .. "s", Color3.fromRGB(220, 170, 255))
	local fim = tick() + real
	while tick() < fim do
		if not ativo or deveParar then return false end
		local restante = math.ceil(fim - tick())
		labelEstado.Text = "Pausa (" .. restante .. "s)"
		task.wait(0.2)
	end
	return true
end

--============================================================
-- PROCESSAR LOCAL
--============================================================
local function processarLocal(pos, nome, cor)
	if not ativo or deveParar then return end
	labelAlvo.Text = string.format("Alvo: %s (%.0f, %.0f, %.0f)", nome, pos.X, pos.Y, pos.Z)
	labelEstado.Text = "A caminhar p/ " .. nome:upper()
	output("🚶 → " .. nome:upper(), cor)

	if irAte(pos) then
		labelEstado.Text = "Em " .. nome:upper() .. " - E"
		task.wait(0.3)
		pressionarTecla(Enum.KeyCode.E)
		task.wait(0.3)
		local tempoReal = variacaoStealth(CONFIG.tempoLocal)
		esperarParado(tempoReal)
		output("✅ " .. nome:upper() .. " feito!", cor)

		if nome == "Grass" then CONFIG.grassFeitas = CONFIG.grassFeitas + 1
		else CONFIG.aguaFeitas = CONFIG.aguaFeitas + 1 end
		atualizarLabels()
	else
		output("❌ Falha em " .. nome:upper(), Color3.fromRGB(255, 120, 120))
	end
end

--============================================================
-- LOOP PRINCIPAL
--============================================================
local function iniciarAutoFarm()
	if ativo then return end
	ativo = true
	deveParar = false

	output("▶ AutoFarm iniciado!", Color3.fromRGB(150, 255, 150))
	notificar("AutoFarm", "Iniciado!")
	tocarSom()

	task.spawn(function()
		while ativo and not deveParar do
			CONFIG.ciclosTotais = CONFIG.ciclosTotais + 1
			output("═══ CICLO " .. CONFIG.ciclosTotais .. " ═══", Color3.fromRGB(255, 220, 120))
			atualizarLabels()

			local funcoes = {}

			if CONFIG.ordem == "grass_agua" then
				table.insert(funcoes, {"Grass", GRASS, Color3.fromRGB(180, 255, 180)})
				table.insert(funcoes, {"Água",  WATER, Color3.fromRGB(180, 220, 255)})
			else
				table.insert(funcoes, {"Água",  WATER, Color3.fromRGB(180, 220, 255)})
				table.insert(funcoes, {"Grass", GRASS, Color3.fromRGB(180, 255, 180)})
			end

			for _, info in ipairs(funcoes) do
				if not ativo or deveParar then break end
				local nome, lista, cor = info[1], info[2], info[3]
				for i, pos in ipairs(lista) do
					if not ativo or deveParar then break end
					processarLocal(pos, nome .. " #" .. i, cor)
				end
			end

			if ativo and not deveParar then
				esperarPausa(CONFIG.pausaCiclo)
				notificar("Ciclo completo", "Ciclos: " .. CONFIG.ciclosTotais)
				tocarSom()
			end
		end

		ativo = false
		deveParar = false
		labelEstado.Text = "Parado"
	end)
end

local function pararAutoFarm()
	deveParar = true
	ativo = false
	liberarTodasTeclas()
	local hum = getHumanoid()
	if hum then
		hum.WalkSpeed = 16
		hum.JumpPower = 50
	end
	labelEstado.Text = "Parado"
	output("⏹ Parado pelo utilizador", Color3.fromRGB(255, 150, 100))
	salvarConfig()
end

--============================================================
-- BOTÕES
--============================================================
botaoIniciar.MouseButton1Click:Connect(iniciarAutoFarm)
botaoParar.MouseButton1Click:Connect(pararAutoFarm)

botaoVelMenos.MouseButton1Click:Connect(function()
	CONFIG.velocidadeMov = math.max(CONFIG.velocidadeMin, CONFIG.velocidadeMov - CONFIG.passoVelocidade)
	atualizarLabels()
end)

botaoVelMais.MouseButton1Click:Connect(function()
	CONFIG.velocidadeMov = math.min(CONFIG.velocidadeMax, CONFIG.velocidadeMov + CONFIG.passoVelocidade)
	atualizarLabels()
end)

botaoVelReset.MouseButton1Click:Connect(function()
	CONFIG.velocidadeMov = 16
	atualizarLabels()
end)

botaoPassoMenos.MouseButton1Click:Connect(function()
	CONFIG.passoVelocidade = math.max(1, CONFIG.passoVelocidade - 1)
	output("Passo velocidade: " .. CONFIG.passoVelocidade)
end)

botaoPassoMais.MouseButton1Click:Connect(function()
	CONFIG.passoVelocidade = math.min(50, CONFIG.passoVelocidade + 1)
	output("Passo velocidade: " .. CONFIG.passoVelocidade)
end)

botaoTempoMenos.MouseButton1Click:Connect(function()
	CONFIG.tempoLocal = math.max(CONFIG.tempoLocalMin, CONFIG.tempoLocal - CONFIG.tempoIncremento)
	atualizarLabels()
end)

botaoTempoMais.MouseButton1Click:Connect(function()
	CONFIG.tempoLocal = math.min(CONFIG.tempoLocalMax, CONFIG.tempoLocal + CONFIG.tempoIncremento)
	atualizarLabels()
end)

botaoPausaMenos.MouseButton1Click:Connect(function()
	CONFIG.pausaCiclo = math.max(0, CONFIG.pausaCiclo - CONFIG.pausaIncremento)
	atualizarLabels()
end)

botaoPausaMais.MouseButton1Click:Connect(function()
	CONFIG.pausaCiclo = math.min(CONFIG.pausaCicloMax, CONFIG.pausaCiclo + CONFIG.pausaIncremento)
	atualizarLabels()
end)

botaoModo.MouseButton1Click:Connect(function()
	CONFIG.modoLocomocao = CONFIG.modoLocomocao == "cframe" and "wasd" or "cframe"
	atualizarLabels()
end)

botaoOrdem.MouseButton1Click:Connect(function()
	CONFIG.ordem = CONFIG.ordem == "grass_agua" and "agua_grass" or "grass_agua"
	atualizarLabels()
end)

botaoStealth.MouseButton1Click:Connect(function()
	CONFIG.stealth = not CONFIG.stealth
	atualizarLabels()
end)

botaoAntiAFK.MouseButton1Click:Connect(function()
	CONFIG.antiAFK = not CONFIG.antiAFK
	atualizarLabels()
end)

botaoReconectar.MouseButton1Click:Connect(function()
	CONFIG.autoReconectar = not CONFIG.autoReconectar
	atualizarLabels()
end)

botaoNotif.MouseButton1Click:Connect(function()
	CONFIG.notificacoes = not CONFIG.notificacoes
	CONFIG.somAtivado = CONFIG.notificacoes
	atualizarLabels()
end)

botaoSomEntrada.MouseButton1Click:Connect(function()
	CONFIG.somEntrada = not CONFIG.somEntrada
	atualizarLabels()
	output("🔔 Som de Entrada: " .. (CONFIG.somEntrada and "ON" or "OFF"))
end)

botaoSalvar.MouseButton1Click:Connect(function()
	salvarConfig()
	output("💾 Config guardada!", Color3.fromRGB(150, 255, 150))
end)

botaoCopiarLog.MouseButton1Click:Connect(function()
	local texto = table.concat(logLinhas, "\n")
	local ok = false
	if setclipboard then pcall(function() setclipboard(texto) ok = true end) end
	if not ok and toclipboard then pcall(function() toclipboard(texto) ok = true end) end
	if ok then
		output("📋 Log copiado!", Color3.fromRGB(150, 255, 150))
	else
		output("⚠️ Clipboard não disponível", Color3.fromRGB(255, 200, 100))
	end
end)

-- Minimizar
local minimizado = false
botaoMinimizar.MouseButton1Click:Connect(function()
	minimizado = not minimizado
	if minimizado then
		Frame.Size = UDim2.new(0, LARGURA, 0, 55)
		botaoMinimizar.Text = "+"
	else
		Frame.Size = UDim2.new(0, LARGURA, 0, ALTURA)
		botaoMinimizar.Text = "−"
	end
end)

-- Tamanho
botaoTamanho.MouseButton1Click:Connect(function()
	if CONFIG.tamanhoGUI == "pequeno" then CONFIG.tamanhoGUI = "medio"
	elseif CONFIG.tamanhoGUI == "medio" then CONFIG.tamanhoGUI = "grande"
	else CONFIG.tamanhoGUI = "pequeno" end

	local t = TAMANHOS[CONFIG.tamanhoGUI]
	LARGURA, ALTURA = t.largura, t.altura
	local alvo = minimizado and UDim2.new(0, LARGURA, 0, 55) or UDim2.new(0, LARGURA, 0, ALTURA)
	TweenService:Create(Frame, TweenInfo.new(0.3), {Size = alvo}):Play()
	output("📐 Tamanho: " .. CONFIG.tamanhoGUI, Color3.fromRGB(200, 150, 255))
end)

botaoFechar.MouseButton1Click:Connect(function()
	pararAutoFarm()
	salvarConfig()
	ScreenGui:Destroy()
end)

--============================================================
-- ATALHOS
--============================================================
UserInputService.InputBegan:Connect(function(input, processado)
	if processado then return end
	if input.KeyCode == Enum.KeyCode.F1 then
		if ativo then pararAutoFarm() else iniciarAutoFarm() end
	elseif input.KeyCode == Enum.KeyCode.F2 then
		botaoMinimizar.MouseButton1Click:Fire()
	elseif input.KeyCode == Enum.KeyCode.F3 then
		botaoModo.MouseButton1Click:Fire()
	end
end)

--============================================================
-- RESPAWN
--============================================================
player.CharacterAdded:Connect(function()
	if ativo then
		task.wait(3)
		output("🔄 Personagem respawnou", Color3.fromRGB(255, 220, 120))
	end
end)

--============================================================
-- INIT
--============================================================
atualizarLabels()
output("✅ AutoFarm Premium v2.3 carregado!", Color3.fromRGB(150, 255, 150))
output("🌾 Grass: " .. #GRASS .. " | 💧 Água: " .. #WATER, Color3.fromRGB(255, 220, 120))
output("⌨️ F1=Iniciar/Parar | F2=Minimizar | F3=Modo", Color3.fromRGB(200, 220, 255))
output("🖱️ Anti-AFK: rato circular a cada 60s", Color3.fromRGB(150, 200, 255))
output("🔔 Som de entrada: " .. (CONFIG.somEntrada and "ON" or "OFF"), Color3.fromRGB(150, 230, 255))

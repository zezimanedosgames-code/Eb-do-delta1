--[[
    EB DO DELTA — SISTEMA DE DELTACOIN (Servidor)
    Instalar em: ServerScriptService (Script)
    
    Funcionalidades:
    - Adiciona 6000 DeltaCoin por clique
    - Cooldown de 5 segundos entre saques
    - Validação total no servidor (cliente nunca decide)
    - Log de auditoria no console
    - Compatível com leaderstats existente (nome: "DeltaCoin")
--]]

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

-- ═══════════════════════════════════════════════════════════
-- CONFIGURAÇÃO
-- ═══════════════════════════════════════════════════════════
local CONFIG = {
    VALOR = 6000,              -- quantia entregue por saque
    COOLDOWN = 5,              -- segundos entre cada saque
    NOME_MOEDA = "DeltaCoin",  -- nome exato da moeda no leaderstats
    MAX_SAQUES_SESSAO = 500,   -- limite de segurança por sessão
}
-- ═══════════════════════════════════════════════════════════

local cooldowns = {}
local contagem = {}

-- Cria o RemoteEvent se não existir
local remote = ReplicatedStorage:FindFirstChild("EBDeltaCoinRemote")
if not remote then
    remote = Instance.new("RemoteEvent")
    remote.Name = "EBDeltaCoinRemote"
    remote.Parent = ReplicatedStorage
end

-- Garante que o leaderstats e o DeltaCoin existem
local function garantirMoeda(player)
    local ls = player:FindFirstChild("leaderstats")
    if not ls then
        ls = Instance.new("Folder")
        ls.Name = "leaderstats"
        ls.Parent = player
    end
    local moeda = ls:FindFirstChild(CONFIG.NOME_MOEDA)
    if not moeda then
        moeda = Instance.new("IntValue")
        moeda.Name = CONFIG.NOME_MOEDA
        moeda.Value = 0
        moeda.Parent = ls
    end
    return moeda
end

-- Handler principal
remote.OnServerEvent:Connect(function(player, acao)
    -- 1) Validação básica
    if typeof(acao) ~= "string" or acao ~= "sacar" then
        return
    end

    -- 2) Cooldown de 5 segundos
    local agora = os.clock()
    local ultimo = cooldowns[player.UserId]
    if ultimo and (agora - ultimo) < CONFIG.COOLDOWN then
        local restante = math.ceil(CONFIG.COOLDOWN - (agora - ultimo))
        remote:FireClient(player, "erro", ("Aguarde %ds para sacar novamente."):format(restante))
        return
    end

    -- 3) Limite de segurança por sessão
    contagem[player.UserId] = (contagem[player.UserId] or 0) + 1
    if contagem[player.UserId] > CONFIG.MAX_SAQUES_SESSAO then
        remote:FireClient(player, "erro", "Limite da sessão atingido.")
        return
    end

    -- 4) Verifica se a moeda existe
    local moeda = garantirMoeda(player)
    if not moeda:IsA("IntValue") then
        remote:FireClient(player, "erro", "Sistema de moeda inválido.")
        return
    end

    -- 5) Aplica o valor (SERVIDOR decide)
    moeda.Value = moeda.Value + CONFIG.VALOR
    cooldowns[player.UserId] = agora

    -- 6) Resposta ao cliente
    remote:FireClient(player, "sucesso", CONFIG.VALOR)

    -- 7) Log de auditoria
    print(("[EB-DELTACOIN] %s (%d) recebeu +%d DeltaCoin (total: %d)"):format(
        player.Name, player.UserId, CONFIG.VALOR, moeda.Value))
end)

-- Setup
Players.PlayerAdded:Connect(garantirMoeda)
for _, p in ipairs(Players:GetPlayers()) do garantirMoeda(p) end

Players.PlayerRemoving:Connect(function(p)
    cooldowns[p.UserId] = nil
    contagem[p.UserId] = nil
end)

print("[EB-DELTACOIN] Servidor inicializado.")
print(("[EB-DELTACOIN] Valor: %d | Cooldown: %ds | Moeda: %s"):format(
    CONFIG.VALOR, CONFIG.COOLDOWN, CONFIG.NOME_MOEDA))--[[
    EB DO DELTA — SISTEMA DE DELTACOIN (Cliente)
    Instalar em: StarterPlayer/StarterPlayerScripts
    
    UI profissional, arrastável, com cooldown visual e notificações.
--]]

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
local remote = ReplicatedStorage:WaitForChild("EBDeltaCoinRemote")

local COOLDOWN = 5 -- deve ser igual ao do servidor

local C = {
    bg=Color3.fromRGB(18,18,24), bgSec=Color3.fromRGB(26,26,34),
    bgCard=Color3.fromRGB(32,32,42), accent=Color3.fromRGB(120,80,255),
    accentGlow=Color3.fromRGB(150,110,255), success=Color3.fromRGB(60,210,120),
    danger=Color3.fromRGB(240,70,90), text=Color3.fromRGB(240,240,250),
    textDim=Color3.fromRGB(160,160,180), border=Color3.fromRGB(60,60,80),
}

local function criar(cl,p,f) local o=Instance.new(cl); for k,v in pairs(p or {}) do o[k]=v end; for _,x in ipairs(f or {}) do x.Parent=o end; return o end
local function cantos(r) return criar("UICorner",{CornerRadius=UDim.new(0,r or 10)}) end
local function grad(a,b,r) return criar("UIGradient",{Color=ColorSequence.new(a,b),Rotation=r or 0}) end

-- GUI raiz
local gui = criar("ScreenGui",{Name="EB_DeltaCoinUI",ResetOnSpawn=false,ZIndexBehavior=Enum.ZIndexBehavior.Sibling,IgnoreGuiInset=true,Parent=playerGui})

-- Painel principal
local painel = criar("Frame",{
    Size=UDim2.fromOffset(420,400),Position=UDim2.new(0.5,-210,0.5,-200),
    BackgroundColor3=C.bg,BorderSizePixel=0,Parent=gui,
},{cantos(16),criar("UIStroke",{Color=C.border,Thickness=1,Transparency=0.3})})

-- Cabeçalho
local header = criar("Frame",{Size=UDim2.new(1,0,0,60),BackgroundColor3=C.bgSec,BorderSizePixel=0,Parent=painel},{cantos(16)})
criar("Frame",{Size=UDim2.new(1,0,0,20),Position=UDim2.new(0,0,1,-20),BackgroundColor3=C.bgSec,BorderSizePixel=0,Parent=header})
criar("Frame",{Size=UDim2.new(1,0,0,3),BackgroundColor3=C.accent,BorderSizePixel=0,Parent=header},{grad(C.accent,C.accentGlow)})
criar("TextLabel",{Text="EB DO DELTA",Font=Enum.Font.GothamBlack,TextSize=20,TextColor3=C.text,BackgroundTransparency=1,Position=UDim2.new(0,20,0,12),Size=UDim2.new(0.6,0,0,22),TextXAlignment=Enum.TextXAlignment.Left,Parent=header})
criar("TextLabel",{Text="SISTEMA DE DELTACOIN",Font=Enum.Font.GothamMedium,TextSize=11,TextColor3=C.textDim,BackgroundTransparency=1,Position=UDim2.new(0,20,0,34),Size=UDim2.new(0.6,0,0,16),TextXAlignment=Enum.TextXAlignment.Left,Parent=header})

-- Botões de controle
local function ctrl(nome,px,cor,txt)
    local b = criar("TextButton",{Name=nome,Size=UDim2.fromOffset(30,30),Position=UDim2.new(1,px,0,15),BackgroundColor3=C.bgCard,Text=txt,Font=Enum.Font.GothamBold,TextSize=16,TextColor3=cor,AutoButtonColor=false,Parent=header},{cantos(8),criar("UIStroke",{Color=C.border,Transparency=0.5})})
    b.MouseEnter:Connect(function() TweenService:Create(b,TweenInfo.new(0.15),{BackgroundColor3=cor}):Play(); TweenService:Create(b,TweenInfo.new(0.15),{TextColor3=C.bg}):Play() end)
    b.MouseLeave:Connect(function() TweenService:Create(b,TweenInfo.new(0.15),{BackgroundColor3=C.bgCard}):Play(); TweenService:Create(b,TweenInfo.new(0.15),{TextColor3=cor}):Play() end)
    return b
end
local btnMin = ctrl("Min",-70,C.accentGlow,"—")
local btnClose = ctrl("Close",-35,C.danger,"✕")

-- Corpo
local corpo = criar("Frame",{Size=UDim2.new(1,-40,1,-140),Position=UDim2.new(0,20,0,78),BackgroundTransparency=1,Parent=painel})

-- Título seção
criar("TextLabel",{Text="SISTEMA DE DINHEIRO",Font=Enum.Font.GothamBold,TextSize=14,TextColor3=C.text,BackgroundTransparency=1,Size=UDim2.new(1,0,0,20),TextXAlignment=Enum.TextXAlignment.Left,Parent=corpo})
criar("Frame",{Size=UDim2.new(1,0,0,2),Position=UDim2.new(0,0,0,26),BackgroundColor3=C.accent,BorderSizePixel=0,Parent=corpo},{grad(C.accent,C.bgCard)})

-- Card descritivo
local card = criar("Frame",{Size=UDim2.new(1,0,0,90),Position=UDim2.new(0,0,0,44),BackgroundColor3=C.bgCard,BorderSizePixel=0,Parent=corpo},{cantos(12),criar("UIStroke",{Color=C.border,Transparency=0.6})})
criar("TextLabel",{Text="💎  DeltaCoin — Fonte oficial de renda",Font=Enum.Font.GothamMedium,TextSize=13,TextColor3=C.text,BackgroundTransparency=1,Position=UDim2.new(0,14,0,12),Size=UDim2.new(1,-28,0,20),TextXAlignment=Enum.TextXAlignment.Left,Parent=card})
criar("TextLabel",{Text="Clique no botão para receber 6.000 DeltaCoin.\nCooldown de 5 segundos entre cada saque.\nTudo é validado pelo servidor.",
    Font=Enum.Font.Gotham,TextSize=11,TextColor3=C.textDim,TextWrapped=true,TextYAlignment=Enum.TextYAlignment.Top,BackgroundTransparency=1,
    Position=UDim2.new(0,14,0,36),Size=UDim2.new(1,-28,0,50),TextXAlignment=Enum.TextXAlignment.Left,Parent=card})

-- Botão de saque
local btnSaque = criar("TextButton",{
    Size=UDim2.new(1,0,0,58),Position=UDim2.new(0,0,0,150),
    BackgroundColor3=C.accent,Text="",AutoButtonColor=false,Parent=corpo,
},{cantos(12),grad(C.accent,C.accentGlow,90),criar("UIStroke",{Color=C.accentGlow,Transparency=0.3})})
local btnLabel = criar("TextLabel",{Text="PEGAR 6.000 DELTACOIN",Font=Enum.Font.GothamBold,TextSize=15,TextColor3=C.text,BackgroundTransparency=1,Size=UDim2.new(1,0,1,0),Parent=btnSaque})
btnSaque.MouseEnter:Connect(function() TweenService:Create(btnSaque,TweenInfo.new(0.15),{BackgroundColor3=C.accentGlow}):Play() end)
btnSaque.MouseLeave:Connect(function() TweenService:Create(btnSaque,TweenInfo.new(0.15),{BackgroundColor3=C.accent}):Play() end)

-- Status / cooldown
local status = criar("TextLabel",{Text="",Font=Enum.Font.GothamMedium,TextSize=12,TextColor3=C.textDim,BackgroundTransparency=1,
    Position=UDim2.new(0,0,0,220),Size=UDim2.new(1,0,0,20),TextXAlignment=Enum.TextXAlignment.Center,Parent=corpo})

-- Rodapé
criar("TextLabel",{Text="EB DO DELTA • v1.0",Font=Enum.Font.Gotham,TextSize=10,TextColor3=C.textDim,BackgroundTransparency=1,
    Position=UDim2.new(0,0,1,-22),Size=UDim2.new(1,0,0,16),TextXAlignment=Enum.TextXAlignment.Center,Parent=corpo})

-- Notificações
local notifContainer = criar("Frame",{Size=UDim2.new(0,360,1,-40),Position=UDim2.new(1,-380,0,20),BackgroundTransparency=1,Parent=gui},{criar("UIListLayout",{SortOrder=Enum.SortOrder.LayoutOrder,Padding=UDim.new(0,8),HorizontalAlignment=Enum.HorizontalAlignment.Right,VerticalAlignment=Enum.VerticalAlignment.Top})})
local function notificar(txt,tipo)
    local cor = tipo=="sucesso" and C.success or C.danger
    local card = criar("Frame",{Size=UDim2.new(1,0,0,54),BackgroundColor3=C.bgSec,BorderSizePixel=0,BackgroundTransparency=1},{cantos(10),criar("UIStroke",{Color=cor,Transparency=0.2})})
    criar("Frame",{Size=UDim2.fromOffset(4,54),BackgroundColor3=cor,BorderSizePixel=0,Parent=card},{cantos(10)})
    criar("TextLabel",{Text=txt,Font=Enum.Font.GothamMedium,TextSize=12,TextColor3=C.text,TextWrapped=true,TextXAlignment=Enum.TextXAlignment.Left,BackgroundTransparency=1,Position=UDim2.new(0,16,0,0),Size=UDim2.new(1,-24,1,0),Parent=card})
    card.Parent=notifContainer
    card.Position=UDim2.new(1,60,0,0)
    TweenService:Create(card,TweenInfo.new(0.25),{BackgroundTransparency=0,Position=UDim2.new(0,0,0,0)}):Play()
    task.delay(3,function() TweenService:Create(card,TweenInfo.new(0.3),{BackgroundTransparency=1,Position=UDim2.new(1,60,0,0)}):Play(); task.wait(0.35); card:Destroy() end)
end

-- ===== LÓGICA DE COOLDOWN VISUAL =====
local emCooldown = false
local ocupado = false

local function iniciarCooldown()
    emCooldown = true
    local restante = COOLDOWN
    btnSaque.BackgroundColor3 = C.bgCard
    btnLabel.TextColor3 = C.textDim
    btnLabel.Text = ("AGUARDE %ds..."):format(restante)
    status.Text = "Cooldown ativo — aguarde para sacar novamente."
    status.TextColor3 = C.textDim

    task.spawn(function()
        while restante > 0 do
            task.wait(1)
            restante -= 1
            if restante > 0 then
                btnLabel.Text = ("AGUARDE %ds..."):format(restante)
                status.Text = ("Próximo saque em %ds"):format(restante)
            end
        end
        emCooldown = false
        btnSaque.BackgroundColor3 = C.accent
        btnLabel.TextColor3 = C.text
        btnLabel.Text = "PEGAR 6.000 DELTACOIN"
        status.Text = "Pronto! Clique para sacar."
        status.TextColor3 = C.success
    end)
end

-- Envio de requisição
btnSaque.MouseButton1Click:Connect(function()
    if ocupado then
        notificar("Aguarde, requisição em andamento.", "erro")
        return
    end
    if emCooldown then
        notificar("Aguarde o cooldown terminar.", "erro")
        return
    end

    ocupado = true
    btnLabel.Text = "PROCESSANDO..."
    remote:FireServer("sacar")

    task.delay(5, function()
        if ocupado then
            ocupado = false
            btnLabel.Text = "PEGAR 6.000 DELTACOIN"
            status.Text = "Tempo esgotado. Tente novamente."
            status.TextColor3 = C.danger
        end
    end)
end)

-- Resposta do servidor
remote.OnClientEvent:Connect(function(tipo, dado)
    ocupado = false
    if tipo == "sucesso" then
        status.Text = ("✓ +%d DeltaCoin adicionados!"):format(dado)
        status.TextColor3 = C.success
        notificar(("Sucesso! +%d DeltaCoin"):format(dado), "sucesso")
        iniciarCooldown()
    elseif tipo == "erro" then
        btnLabel.Text = "PEGAR 6.000 DELTACOIN"
        status.Text = "✕ " .. tostring(dado)
        status.TextColor3 = C.danger
        notificar(tostring(dado), "erro")
    end
end)

-- Minimizar / Fechar
local mini = false
btnMin.MouseButton1Click:Connect(function()
    mini = not mini
    TweenService:Create(painel,TweenInfo.new(0.35,Enum.EasingStyle.Quart),{Size=UDim2.fromOffset(420, mini and 60 or 400)}):Play()
    corpo.Visible = not mini
    btnMin.Text = mini and "▢" or "—"
end)
btnClose.MouseButton1Click:Connect(function()
    TweenService:Create(painel,TweenInfo.new(0.25),{Size=UDim2.fromOffset(0,0),Position=UDim2.new(0.5,0,0.5,0)}):Play()
    task.wait(0.3); gui:Destroy()
end)

-- Drag
local dragging,dragStart,startPos
header.InputBegan:Connect(function(i)
    if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then
        dragging=true; dragStart=i.Position; startPos=painel.Position
    end
end)
UserInputService.InputChanged:Connect(function(i)
    if dragging and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then
        local d=i.Position-dragStart
        painel.Position=UDim2.new(startPos.X.Scale,startPos.X.Offset+d.X,startPos.Y.Scale,startPos.Y.Offset+d.Y)
    end
end)
UserInputService.InputEnded:Connect(function(i)
    if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then dragging=false end
end)

-- Animação de entrada
painel.Size = UDim2.fromOffset(0,0); painel.Position = UDim2.new(0.5,0,0.5,0)
TweenService:Create(painel,TweenInfo.new(0.4,Enum.EasingStyle.Back,Enum.EasingDirection.Out),{
    Size=UDim2.fromOffset(420,400), Position=UDim2.new(0.5,-210,0.5,-200)
}):Play()

print("[EB-DELTACOIN] Interface carregada.")

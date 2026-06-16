local Players = game:GetService("Players")
local LP = Players.LocalPlayer

-- Nome do atributo que identifica o Ninja
local NINJA_ATTRIBUTE = "IsNinjaUser"

-- Função para criar a tag
local function createTag(head)
    if head:FindFirstChild("NinjaTagBillboard") then return end

    local billboardGui = Instance.new("BillboardGui")
    billboardGui.Name = "NinjaTagBillboard"
    billboardGui.Size = UDim2.new(0, 150, 0, 50) 
    billboardGui.StudsOffset = Vector3.new(0, 3, 0)
    billboardGui.AlwaysOnTop = true
    billboardGui.LightInfluence = 0
    billboardGui.Parent = head

    local textLabel = Instance.new("TextLabel")
    textLabel.Size = UDim2.new(1, 0, 1, 0)
    textLabel.BackgroundTransparency = 1
    textLabel.Text = "NINJA USER"
    textLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
    textLabel.Font = Enum.Font.GothamBold
    textLabel.TextScaled = true
    textLabel.TextStrokeTransparency = 0
    textLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    textLabel.Parent = billboardGui
end

-- Função para marcar o personagem como Ninja (para que outros te vejam)
local function markAsNinja(character)
    if character then
        character:SetAttribute(NINJA_ATTRIBUTE, true)
        -- Coloca a tag em si mesmo também para confirmação
        local head = character:WaitForChild("Head", 10)
        if head then createTag(head) end
    end
end

-- Função para verificar se alguém é Ninja e colocar a tag
local function checkAndTag(character)
    if not character then return end
    
    -- Verifica se o personagem tem o atributo de Ninja
    if character:GetAttribute(NINJA_ATTRIBUTE) == true then
        local head = character:WaitForChild("Head", 10)
        if head then createTag(head) end
    end

    -- Fica vigiando se o atributo for adicionado depois (caso o outro script demore a carregar)
    character:GetAttributeChangedSignal(NINJA_ATTRIBUTE):Connect(function()
        if character:GetAttribute(NINJA_ATTRIBUTE) == true then
            local head = character:WaitForChild("Head", 10)
            if head then createTag(head) end
        end
    end)
end

-- Configura o Jogador Local (Você)
LP.CharacterAdded:Connect(markAsNinja)
if LP.Character then markAsNinja(LP.Character) end

-- Configura a detecção para os outros jogadores
local function setupPlayerDetection(player)
    player.CharacterAdded:Connect(checkAndTag)
    if player.Character then checkAndTag(player.Character) end
end

Players.PlayerAdded:Connect(setupPlayerDetection)
for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LP then
        setupPlayerDetection(player)
    end
end

print("Sistema de Detecção Ninja Seletiva Ativado.")

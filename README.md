-- Biblioteca e UI
local Library = loadstring(game:HttpGet("https://raw.githubusercontent.com/bloodball/-back-ups-for-libs/main/wizard"))()
local Window = Library:NewWindow("EGO")
local Section = Window:NewSection("Controle")

-- Configurações do drible
getgenv().AutoDribbleSettings = getgenv().AutoDribbleSettings or {
    Enabled = true,
    range = 22,
    reactionTime = 0.025
}
local S = getgenv().AutoDribbleSettings

local R = game:GetService("ReplicatedStorage")
local P = game:GetService("Players")
local U = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local L = P.LocalPlayer or P.PlayerAdded:Wait()

local function initCharacter()
    local C = L.Character or L.CharacterAdded:Wait()
    local H = C:WaitForChild("HumanoidRootPart")
    local M = C:WaitForChild("Humanoid")
    return C, H, M
end

local C, H, M = initCharacter()
L.CharacterAdded:Connect(function()
    C, H, M = initCharacter()
end)

local B = R.Packages.Knit.Services.BallService.RE.Dribble
local A = require(R.Assets.Animations)

local function GetAnimation(style)
    if not A.Dribbles[style] then return nil end
    local anim = Instance.new("Animation")
    anim.AnimationId = A.Dribbles[style]
    return M and M:LoadAnimation(anim)
end

local function IsTryingToSteal(player)
    if player == L then return false end
    local char = player.Character
    if not char then return false end
    local sliding = char:FindFirstChild("Values") and char.Values.Sliding
    return sliding and sliding.Value == true
end

-- Sistema de reconhecimento de equipe por camisa
local SameShirtPlayers = {}
local function identifyTeamByShirt()
    local myChar = L.Character or L.CharacterAdded:Wait()
    local myShirt = myChar:FindFirstChildWhichIsA("Shirt")
    if not myShirt then return end

    for _, player in pairs(P:GetPlayers()) do
        if player ~= L and player.Character then
            local theirShirt = player.Character:FindFirstChildWhichIsA("Shirt")
            if theirShirt and theirShirt.ShirtTemplate == myShirt.ShirtTemplate then
                SameShirtPlayers[player] = true
            end
        end
    end
end

identifyTeamByShirt()
P.PlayerAdded:Connect(function(p)
    p.CharacterAdded:Connect(function()
        task.delay(2, identifyTeamByShirt)
    end)
end)

-- Função para saber se é oponente
local function IsOpponent(player)
    if not player or player == L then return false end
    return not SameShirtPlayers[player]
end

local function IsFacingPlayer(enemyHRP, targetHRP)
    local directionToTarget = (targetHRP.Position - enemyHRP.Position).Unit
    local enemyLookVector = enemyHRP.CFrame.LookVector
    local dot = directionToTarget:Dot(enemyLookVector)
    return dot > 0.5
end

local function Dribble(dist)
    if not S.Enabled or not C.Values.HasBall.Value then return end
    B:FireServer()
    local style = L.PlayerStats.Style.Value
    local anim = GetAnimation(style)
    if anim then
        anim:Play()
        anim:AdjustSpeed(math.clamp(1 + (10 - dist) / 10, 1, 2))
    end
    local ball = workspace:FindFirstChild("Football")
    if ball then
        ball.AssemblyLinearVelocity = Vector3.new()
        ball.CFrame = C.HumanoidRootPart.CFrame * CFrame.new(0, -2.5, -1.5)
    end
end

-- Loop principal
U.Heartbeat:Connect(function()
    if not S.Enabled or not C or not H then return end

    for _, p in pairs(P:GetPlayers()) do
        if IsOpponent(p) and IsTryingToSteal(p) then
            local enemyHRP = p.Character and p.Character:FindFirstChild("HumanoidRootPart")
            if enemyHRP then
                local dist = (enemyHRP.Position - H.Position).Magnitude
                if dist < S.range and IsFacingPlayer(enemyHRP, H) then
                    local adjustedReaction = math.clamp(S.reactionTime * (dist / S.range), 0.001, S.reactionTime)
                    task.delay(adjustedReaction, function()
                        if C and C:FindFirstChild("Values") and C.Values:FindFirstChild("HasBall") and C.Values.HasBall.Value then
                            Dribble(dist)
                        end
                    end)
                end
            end
        end
    end
end)

-- Botão para ativar/desativar o Auto Drible
Section:CreateToggle("Auto Drible", function(state)
    S.Enabled = state
end)

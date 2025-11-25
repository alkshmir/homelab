# GPT Bot

Discord bot powered by OpenAI for Kubernetes deployment.

## Preparation

### Create Secret

1. Copy the secret example file:
   ```bash
   cp secret-example.yaml secret.yaml
   ```

2. Edit `secret.yaml` and replace the placeholder values:
   - `<OPENAI API KEY>`: Your OpenAI API key from https://platform.openai.com/api-keys
   - `<DISCORD BOT TOKEN>`: Your Discord bot token from https://discord.com/developers/applications

3. Create the namespace and apply the secret:
   ```bash
   kubectl create namespace gpt-bot
   kubectl apply -f secret.yaml
   ```

## Deployment

```bash
kubectl apply -f gpt-bot.yaml
```

## Verification

Check if the bot is running:

```bash
kubectl get pods -n gpt-bot
kubectl logs -n gpt-bot -l app=gpt-bot
```

## Reference

https://github.com/alkshmir/go-openai-discord

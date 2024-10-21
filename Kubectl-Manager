#!/bin/bash

# Default namespace
NAMESPACE="default"  # Change this if you need a different default namespace

# Directory where environment YAML files are stored
KUBE_CONFIG_DIR="$HOME/.kube"

# Helper function to display usage
usage() {
  echo "Toolname - A Kubernetes Pod and Environment Management Tool"
  echo
  echo "Usage: $0 -a <app-name> [-n <namespace>] -m <bash|rails|set-secret|get-secret|switch-env] [-e <env-var>] [-v <value>]"
  echo
  echo "Options:"
  echo "  -a <app-name>       : The application name (required, except for switch-env mode)."
  echo "  -n <namespace>      : The Kubernetes namespace (optional, default: $NAMESPACE)."
  echo "  -m <bash|rails|set-secret|get-secret|switch-env>  : The mode to run (required):"
  echo "                         bash: Opens a bash session inside the app's pod."
  echo "                         rails: Opens the Rails console inside the app's pod."
  echo "                         get-secret: Fetches a secret from Kubernetes Secrets."
  echo "                         set-secret: Sets or updates a secret in Kubernetes Secrets."
  echo "                         switch-env: Switches between Kubernetes YAML environments."
  echo "  -e <env-var>        : The environment variable name (required for get-env, set-env, and get-secret modes)."
  echo "  -v <value>          : The value to set for the environment variable (required for set-env and set-secret modes)."
  echo "  --help              : Display this help message."
  echo
  echo "Examples:"
  echo "  1. Open a bash session in the app pod:"
  echo "     $0 -a my-app -m bash"
  echo
  echo "  2. Open a Rails console in the app pod:"
  echo "     $0 -a my-app -m rails"
  echo
  echo "  3. Fetch a secret from Kubernetes Secrets:"
  echo "     $0 -a my-app -m get-secret -e SECRET_KEY_BASE"
  echo
  echo "  4. Set a secret in Kubernetes Secrets:"
  echo "     $0 -a my-app -m set-secret -e SECRET_KEY_BASE -v new_secret_value"
  echo
  echo "  5. Switch Kubernetes environment:"
  echo "     $0 -m switch-env"
  exit 1
}

# Display help message if --help is provided
if [[ "$1" == "--help" ]]; then
  usage
fi

# Parse command-line options
while getopts ":a:n:m:e:v:" opt; do
  case $opt in
    a) APP_NAME="$OPTARG" ;;
    n) NAMESPACE="$OPTARG" ;;
    m) MODE="$OPTARG" ;;
    e) ENV_VAR="$OPTARG" ;;
    v) ENV_VALUE="$OPTARG" ;;
    *) usage ;;
  esac
done

# Function to handle environment switching
switch_env() {
  echo "Available environments:"
  i=1
  declare -A env_map

  # List available YAML files in the .kube directory
  for file in $KUBE_CONFIG_DIR/*.yaml; do
    env_name=$(basename "$file" .yaml)
    echo "$i) $env_name"
    env_map[$i]=$file
    ((i++))
  done

  # Prompt the user to select an environment
  read -p "Select the environment (enter the number): " env_choice

  # Validate the selection
  if [[ -z "${env_map[$env_choice]}" ]]; then
    echo "Invalid selection. Exiting..."
    exit 1
  fi

  # Set the selected environment
  SELECTED_ENV_FILE="${env_map[$env_choice]}"
  export KUBECONFIG="$SELECTED_ENV_FILE"
  echo $KUBECONFIG
  cp $(echo $KUBECONFIG) ~/.kube/config
  echo "Switched to environment: $(basename "$SELECTED_ENV_FILE")"
}

# Perform operations based on the mode
case $MODE in
  switch-env)
    switch_env
    exit 0
    ;;
esac

# Validate required arguments (skip if switch-env is selected)
if [ "$MODE" != "switch-env" ] && ([ -z "$APP_NAME" ] || [ -z "$MODE" ]); then
  usage
fi

# Validate mode (bash, rails, get-env, set-env, get-secret)
if [ "$MODE" != "bash" ] && [ "$MODE" != "rails" ] && [ "$MODE" != "get-secret" ] && [ "$MODE" != "set-secret" ] && [ "$MODE" != "switch-env" ] && [ "$MODE" != "get-branch" ]; then
  echo "Error: Invalid mode '$MODE'. Valid modes are 'bash', 'rails', 'get-env', 'set-env', 'get-secret', or 'switch-env'."
  usage
fi

# Set the app label using the provided app name
APP_LABEL="app=${APP_NAME}"

# Get the pod name for the app
POD_NAME=$(kubectl get pods -n "$NAMESPACE" -l "$APP_LABEL" -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)

# Check if a pod was found
if [ -z "$POD_NAME" ]; then
  echo "Error: No pod found for app '$APP_NAME' in namespace '$NAMESPACE'."
  exit 1
fi

# Perform operations based on the mode
case $MODE in
  bash)
    # Check if additional command is provided
    if [ "$#" -gt "$OPTIND" ]; then
      CMD="${@:OPTIND}"
      echo "Executing command '$CMD' in bash session on pod $POD_NAME for app $APP_NAME in namespace $NAMESPACE..."
      kubectl exec -it "$POD_NAME" -n "$NAMESPACE" -- bash -c "$CMD"
    else
      echo "Opening bash session on pod $POD_NAME for app $APP_NAME in namespace $NAMESPACE..."
      kubectl exec -it "$POD_NAME" -n "$NAMESPACE" -- bash
    fi
    ;;
  
  rails)
    echo "Opening Rails console on pod $POD_NAME for app $APP_NAME in namespace $NAMESPACE..."
    kubectl exec -it "$POD_NAME" -n "$NAMESPACE" -- bash -c "rails c"
    ;;
  
  get-secret)
    if [ -z "$ENV_VAR" ]; then
      echo "Error: You must provide an environment variable with -e for get-secret mode."
      usage
    fi
    echo "Fetching environment variable '$ENV_VAR' from secret '$APP_NAME' in namespace $NAMESPACE..."
    kubectl get secret "$APP_NAME" -n "$NAMESPACE" -o go-template='{{ range $key, $value := .data }}{{ printf "%s=%s\n" $key (printf "%s" $value | base64decode) }}{{ end }}' | grep "$ENV_VAR"
    ;;
    
  set-secret)
    if [ -z "$ENV_VAR" ] || [ -z "$ENV_VALUE" ]; then
      echo "Error: You must provide an environment variable with -e and a value with -v for set-secret mode."
      usage
    fi
    echo "Setting environment variable '$ENV_VAR' to '$ENV_VALUE' in secret '$APP_NAME' in namespace $NAMESPACE..."
    kubectl patch secret "$APP_NAME" -n "$NAMESPACE" --type merge -p "{\"stringData\":{\"$ENV_VAR\":\"$ENV_VALUE\"}}"
    ;;
  
  get-branch)
    echo "Fetching deployed branch from pod $POD_NAME for app $APP_NAME in namespace $NAMESPACE..."

    # Step 1: Use git to get the branch name that contains the HEAD
    BRANCH_NAME=$(kubectl exec -it "$POD_NAME" -n "$NAMESPACE" -- bash -c "git branch -r --contains HEAD")

    if [ -z "$BRANCH_NAME" ]; then
      echo "Error: Could not find the branch containing the current HEAD commit."
      exit 1
    fi

    echo "Deployed branch: $BRANCH_NAME"
    ;;
  
  *)
    usage
    ;;
esac

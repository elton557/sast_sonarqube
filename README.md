# Estágio de Verificação SonarQube

## Visão Geral

Este projeto implementa um estágio de **SonarQube** no pipeline CI/CD do GitLab. O estágio `sonarqube_check` realiza uma análise estática do código para garantir a qualidade e segurança antes da implantação, enviando um e-mail com o resultado e o link do relatório.

## Funcionalidades Principais

- **SonarQube**: Ferramenta open-source para análise de qualidade de código.
- **Integração CI/CD GitLab**: Verificações contínuas de qualidade de código durante o ciclo de vida de desenvolvimento.
- **Envio de E-mails**: Resultado da análise é enviado automaticamente para um destinatário.
- **Suporte a Proxy**: Uso de proxy para baixar dependências e conectar ao SonarQube.

## Requisitos

### Variáveis de Ambiente

Certifique-se de configurar as seguintes variáveis de ambiente no GitLab:

#### Proxy:

- `PROXY_DOCKER`: URL do proxy HTTP para o Docker.

#### Configurações SonarQube:

- `SONAR_HOST_URL`: URL do servidor SonarQube.
- `SONAR_LOGIN`: Token de autenticação do SonarQube.
- `SONAR_PROJECT_KEY`: Chave do projeto no SonarQube.

#### Configurações SMTP (para envio de e-mail):

- `HOST_SMTP_ZAP`: Endereço do servidor SMTP.
- `USER_SMTP_ZAP`: Nome de usuário para autenticação no SMTP.
- `PASSWORD_ZAP`: Senha de autenticação no SMTP.
- `REMETENTE_EMAIL_ZAP`: E-mail do destinatário para o envio do resultado da análise.

## Configuração do Pipeline

Aqui está o arquivo `.gitlab-ci.yml` para configurar o estágio `sonarqube_check`:

```yaml
sonarqube_check:
  stage: sonarqube_check   # Estágio que faz a checagem de vulnerabilidades no SonarQube
  before_script: []
  services:
    - docker:stable-dind
  image:
    name: sonarsource/sonar-scanner-cli:5.0.1   # Imagem do SonarQube usada
    entrypoint: [""]
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"  # Diretório para cache da análise
    GIT_DEPTH: "0"                               # Instruir o Git a buscar todos os branches
    SONAR_PROJECT_KEY: "${CI_PROJECT_NAME}"      # Nome do projeto no SonarQube
  cache:
    key: "${CI_JOB_NAME}"
    paths:
      - .sonar/cache                             # Cache do SonarQube
  script:
    - export http_proxy=$PROXY_DOCKER
    - export https_proxy=$PROXY_DOCKER
    - apk --no-cache add --allow-untrusted msmtp
    - cp /root/.docker/certs/sonarqube.crt .
    # Importar o certificado do SonarQube para o Java
    - keytool -importcert -file sonarqube.crt -cacerts -alias sonarqube.crt -storepass changeit -noprompt
    # Comando que faz o scanner no projeto e grava a saída no log
    - sonar-scanner -Dsonar.projectKey=${SONAR_PROJECT_KEY} -Dsonar.projectName=${SONAR_PROJECT_NAME} -Dsonar.sources=application -Dsonar.host.url=${SONAR_HOST_URL} -Dsonar.login=${SONAR_LOGIN} -Dsonar.exclusions=node_modules/**,dist/**,coverage/** | tee build.log

    # Verifica o status da análise
    - |
      if grep -q 'ANALYSIS SUCCESSFUL, you can find the results at: ' build.log; then
        echo "ANALYSIS SUCCESSFUL" > result.txt;
      else
        echo "ANALYSIS FAILED" > result.txt;
      fi

    # Obtém a URL do dashboard do SonarQube
    - export SONAR_DASHBOARD_URL=$(grep -oE 'https?://sonarqube.teste.com.br/\S+' build.log | head -1)

    # Configurações para envio de e-mail
    - |
      current_time=$(date "+%Y-%m-%d")
      echo "account gitlab" > /etc/msmtprc
      echo "host $HOST_SMTP_ZAP" >> /etc/msmtprc
      echo "port 587" >> /etc/msmtprc
      echo "auth on" >> /etc/msmtprc
      echo "user $USER_SMTP_ZAP" >> /etc/msmtprc
      echo "password $PASSWORD_ZAP" >> /etc/msmtprc
      echo "tls on" >> /etc/msmtprc
      echo "tls_starttls on" >> /etc/msmtprc
      echo "tls_certcheck off" >> /etc/msmtprc
      echo "from $USER_SMTP_ZAP" >> /etc/msmtprc
      echo "logfile /var/log/msmtp.log" >> /etc/msmtprc
      echo "account default: gitlab" >> /etc/msmtprc
    - chmod 600 /etc/msmtprc

    # Envia o e-mail com base no resultado da análise
    - |
      if grep -q "ANALYSIS FAILED" result.txt; then
        {
          echo "Subject: SonarQube Analysis Failed for $current_time - $CI_PROJECT_NAME"
          echo "To: $REMETENTE_EMAIL_ZAP"
          echo "Content-Type: text/plain; charset=\"utf-8\""
          echo ""
          echo "Olá,"
          echo ""
          echo "A análise do SonarQube falhou. Você pode verificar os detalhes no seguinte link:"
          echo "$SONAR_DASHBOARD_URL"
          echo ""
          echo "Atenciosamente,"
        } | msmtp -a gitlab -t $REMETENTE_EMAIL_ZAP
        exit 1  # Força a falha do job após o envio do e-mail
      else
        {
          echo "Subject: SonarQube Analysis Successful for $current_time - $CI_PROJECT_NAME"
          echo "To: $REMETENTE_EMAIL_ZAP"
          echo "Content-Type: text/plain; charset=\"utf-8\""
          echo ""
          echo "Olá,"
          echo ""
          echo "A análise do SonarQube foi concluída com sucesso. Você pode acessar o resultado no seguinte link:"
          echo "$SONAR_DASHBOARD_URL"
          echo ""
          echo "Atenciosamente,"
        } | msmtp -a gitlab -t $REMETENTE_EMAIL_ZAP
      fi

  allow_failure: true   # Essa tarefa pode falhar, e a pipeline vai seguir seu fluxo
  only:
    - master
  tags:
    - docker            # Define que o job deve ser executado em runners etiquetados com 'docker'

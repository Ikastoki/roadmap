# Integracion con pipelines CI/CD

Un repositorio de infraestructura como codigo tiene sentido solo si los cambios fluyen a traves de un proceso de validacion automatica antes de llegar al entorno de produccion. Integrar IaC en un pipeline de CI/CD transforma cada modificacion de archivos HCL en una secuencia verificada: formato, sintaxis, planificacion y aplicacion controlada. Este enfoque elimina los errores humanos mas comunes (sintaxis incorrecta, variables no definidas, dependencias circulares) y garantiza que cada cambio este documentado, revisado y aprobado antes de ejecutarse contra la nube.

El pipeline clasico para Terraform se divide en cuatro fases distintas, cada una con un proposito especifico y un nivel de permisos diferente sobre los recursos de la nube. Comprender esta separacion es clave para disenar pipelines seguros: la fase de linting no necesita credenciales de cloud, la fase de plan necesita credenciales de solo lectura y la fase de apply necesita credenciales de escritura completas.

## Fase de formato y validacion

La primera barrera del pipeline verifica que los archivos cumplan con los estilos de codigo establecidos y que la sintaxis sea correcta. `terraform fmt` reescribe los archivos para ajustarlos al formato canonico de HCL, y `terraform validate` verifica que la configuracion sea sintacticamente valida y que todas las referencias entre recursos sean coherentes.

```yaml
# GitHub Actions - Fase de linting
name: Terraform Lint
on:
  pull_request:
    paths:
      - '**/*.tf'
      - '**/*.tfvars'

jobs:
  format-validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.0"

      - name: Check formatting
        run: terraform fmt -check -recursive -diff

      - name: Validate all modules
        run: |
          for dir in $(find . -type f -name "*.tf" -exec dirname {} \; | sort -u); do
            echo "Validating $dir"
            cd "$dir" && terraform init -backend=false && terraform validate
            cd -
          done
```

> `terraform fmt -check` no modifica archivos; devuelve un codigo de salida distinto de cero si alguno no cumple el formato. Esto es ideal para pipelines donde el objetivo es detectar, no corregir. Si se ejecuta `terraform fmt -recursive` sin `-check`, los archivos se reescriben automaticamente.

## Fase de planificacion

La fase de planificacion ejecuta `terraform plan` y genera un diff detallado de los cambios propuestos. Este diff se puede publicar como comentario en el pull request, permitiendo a los revisores entender exactamente que recursos seran afectados antes de aprobar el cambio.

```yaml
# GitHub Actions - Fase de plan
  plan:
    runs-on: ubuntu-latest
    needs: format-validate
    permissions:
      contents: read
      pull-requests: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.0"

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/terraform-plan
          aws-region: eu-west-1

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan 2>&1 | tee plan-output.txt

      - name: Comment plan on PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('plan-output.txt', 'utf8');
            const truncated = plan.length > 60000
              ? plan.substring(0, 60000) + '\n\n... (plan truncado)'
              : plan;
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `### Plan de Terraform\n\`\`\`\n${truncated}\n\`\`\``
            });
```

El rol de IAM utilizado en esta fase tiene permisos de solo lectura sobre los servicios que Terraform gestiona. Esto es una restriccion de seguridad: incluso si el pipeline es comprometido, el rol de planificacion no puede crear ni eliminar recursos reales.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "rds:Describe*",
        "s3:GetBucketLocation",
        "s3:ListBucket",
        "iam:GetRole",
        "iam:GetPolicy"
      ],
      "Resource": "*"
    }
  ]
}
```

## Fase de aplicacion

La fase de apply solo se ejecuta cuando el pull request ha sido aprobado y fusionado a la rama principal. Este patron se conoce como "merge-to-apply" y es el modelo mas seguro porque aprovecha los mecanismos nativos de proteccion de ramas de GitHub (branch protection rules, required reviewers) para controlar cuando se modifican los recursos reales.

```yaml
# GitHub Actions - Fase de apply
  apply:
    runs-on: ubuntu-latest
    needs: plan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production
    concurrency:
      group: terraform-apply
      cancel-in-progress: false
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.0"

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/terraform-apply
          aws-region: eu-west-1

      - name: Terraform Init
        run: terraform init

      - name: Terraform Apply
        run: terraform apply -auto-approve

      - name: Post apply notification
        if: success()
        run: |
          echo "Aplicacion completada. SHA: ${{ github.sha }}"
          echo "Autor: ${{ github.actor }}"
```

> El bloque `concurrency` con `cancel-in-progress: false` es critico: si dos aplicaciones se ejecutan simultaneamente sobre el mismo estado de Terraform, una de ellas puede corromper el state o aplicar cambios sobre una base que ya no es la esperada. El lock serializa las ejecuciones de apply en la rama principal.

## Gestion de planes entre fases

En escenarios complejos donde el plan se ejecuta en un pull request y el apply se ejecuta despues de fusionar, existe un problema de consistencia: el plan podria no ser valido despues de que otros cambios se hayan fusionado a la rama principal simultaneamente. Para resolver esto, el plan serializado se almacena como un artifact del pipeline y se recupera durante la fase de apply.

```yaml
# Artifact sharing entre plan y apply
      - name: Upload plan artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: tfplan
          retention-days: 5

# En el job de apply:
      - name: Download plan artifact
        uses: actions/download-artifact@v4
        with:
          name: tfplan

      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan
```

Cuando se aplica un plan serializado, Terraform verifica que el estado actual de los recursos coincida con lo que el plan esperaba encontrar. Si hay divergencias (por ejemplo, otro cambio ya fue aplicado), Terraform aborta la operacion y el pipeline falla, evitando aplicaciones parciales o incoherentes.

```yaml
# Pipeline completo simplificado
name: Terraform CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform fmt -check -recursive
      - run: |
          for dir in $(find . -name "*.tf" -exec dirname {} \; | sort -u); do
            (cd "$dir" && terraform init -backend=false && terraform validate)
          done

  plan:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_PLAN_ROLE }}
          aws-region: eu-west-1
      - run: terraform init
      - run: terraform plan -no-color -out=tfplan 2>&1
      - uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: tfplan

  apply:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    needs: plan
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_APPLY_ROLE }}
          aws-region: eu-west-1
      - run: terraform init
      - uses: actions/download-artifact@v4
        with:
          name: tfplan
      - run: terraform apply -auto-approve tfplan
```


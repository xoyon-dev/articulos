---


---

<h1 id="hello-terraform-part-2-state-en-gitlab-y-conexión-keyless-con-aws-oidc">Hello Terraform Part 2: State en GitLab y Conexión Keyless con AWS (OIDC)</h1>
<p>En la <a href="https://medium.com/@menchuxoyon/tu-primer-deploy-con-terraform-aws-gitlab-ci-cd-hello-world-5c06d914191b">primera parte</a> desplegamos un sitio estático en S3 con Terraform y GitLab CI/CD. Funciona, pero tiene dos problemas:</p>
<ul>
<li>
<p>El <strong>state de Terraform</strong> vive en tu máquina. Si lo perdés, Terraform no sabe qué recursos existen.</p>
</li>
<li>
<p>Las <strong>access keys de AWS</strong> son estáticas y están guardadas en GitLab como secretos. Si se filtran, alguien tiene acceso permanente a tu cuenta.</p>
</li>
</ul>
<p>En esta parte solucionamos ambos. Sin crear recursos extra en AWS para el state, y eliminando las access keys por completo.</p>
<hr>
<h2 id="requisitos">Requisitos</h2>
<p>Todo lo de la parte 1 funcionando:</p>
<ul>
<li>
<p>Bucket S3 con el sitio estático</p>
</li>
<li>
<p>Pipeline de GitLab desplegando con <code>aws s3 sync</code></p>
</li>
<li>
<p>Usuario <code>gitlab-deployer</code> con access keys</p>
</li>
</ul>
<hr>
<h2 id="part-1-—-state-en-gitlab-http-backend">Part 1 — State en GitLab HTTP Backend</h2>
<p>GitLab tiene un backend HTTP integrado para almacenar el state de Terraform. No necesitás S3 ni DynamoDB — GitLab se encarga del almacenamiento y del locking.</p>
<h3 id="paso-1.1-—-editá-inframain.tf">Paso 1.1 — Editá <code>infra/main.tf</code></h3>
<p>Reemplazá el bloque <code>terraform {}</code>:</p>
<pre class=" language-hcl"><code class="prism  language-hcl">
terraform {

required_providers {

aws = {

source = "hashicorp/aws"

version = "~&gt; 5.0"

}

}

  

backend "http" {

}

}

</code></pre>
<p>El backend queda vacío porque los parámetros se pasan por variables de entorno.</p>
<h3 id="paso-1.2-—-creá-un-personal-access-token">Paso 1.2 — Creá un Personal Access Token</h3>
<p>Lo necesitás para migrar el state desde tu máquina local.</p>
<ol>
<li>
<p>En GitLab → <strong>User Settings → Access Tokens</strong></p>
</li>
<li>
<p>Nombre: <code>terraform-state</code></p>
</li>
<li>
<p>Scope: <code>api</code></p>
</li>
<li>
<p>Crealo y copiá el token</p>
</li>
</ol>
<h3 id="paso-1.3-—-obtenté-tu-project-id">Paso 1.3 — Obtenté tu Project ID</h3>
<p>En tu repo → <strong>Settings → General</strong>, aparece arriba de todo. Es un número (ej: <code>79496053</code>).</p>
<h3 id="paso-1.4-—-migrá-el-state-local-a-gitlab">Paso 1.4 — Migrá el state local a GitLab</h3>
<pre class=" language-bash"><code class="prism  language-bash">
<span class="token function">cd</span>  infra

  

terraform  init  \

-backend-config<span class="token operator">=</span><span class="token string">"address=https://gitlab.com/api/v4/projects/TU_PROJECT_ID/terraform/state/hello-terraform"</span> \

-backend-config<span class="token operator">=</span><span class="token string">"lock_address=https://gitlab.com/api/v4/projects/TU_PROJECT_ID/terraform/state/hello-terraform/lock"</span>  \

-backend-config<span class="token operator">=</span><span class="token string">"unlock_address=https://gitlab.com/api/v4/projects/TU_PROJECT_ID/terraform/state/hello-terraform/lock"</span> \

-backend-config<span class="token operator">=</span><span class="token string">"username=TU_USUARIO_GITLAB"</span>  \

-backend-config<span class="token operator">=</span><span class="token string">"password=TU_PERSONAL_ACCESS_TOKEN"</span> \

-migrate-state

</code></pre>
<p>Terraform te pregunta si querés copiar el state. Decí <code>yes</code>.</p>
<h3 id="paso-1.5-—-verificá-que-se-subió">Paso 1.5 — Verificá que se subió</h3>
<p>Andá a tu repo → <strong>Operate → Terraform states</strong>. Deberías ver <code>hello-terraform</code> listado ahí.</p>
<h3 id="paso-1.6-—-borrá-el-state-local">Paso 1.6 — Borrá el state local</h3>
<pre class=" language-bash"><code class="prism  language-bash">
<span class="token function">rm</span>  -f  terraform.tfstate  terraform.tfstate.backup

</code></pre>
<h3 id="paso-1.7-—-actualizá-el-pipeline">Paso 1.7 — Actualizá el pipeline</h3>
<p>El pipeline necesita las variables del backend para poder leer/escribir el state. Editá el stage <code>validate</code> en <code>.gitlab-ci.yml</code>:</p>
<pre class=" language-yaml"><code class="prism  language-yaml">
<span class="token key atrule">validate</span><span class="token punctuation">:</span>

<span class="token key atrule">stage</span><span class="token punctuation">:</span> validate

<span class="token key atrule">image</span><span class="token punctuation">:</span>

<span class="token key atrule">name</span><span class="token punctuation">:</span> hashicorp/terraform<span class="token punctuation">:</span><span class="token number">1.7</span>

<span class="token key atrule">entrypoint</span><span class="token punctuation">:</span> <span class="token punctuation">[</span><span class="token string">""</span><span class="token punctuation">]</span>

<span class="token key atrule">variables</span><span class="token punctuation">:</span>

<span class="token key atrule">TF_HTTP_ADDRESS</span><span class="token punctuation">:</span> <span class="token string">"https://gitlab.com/api/v4/projects/${CI_PROJECT_ID}/terraform/state/hello-terraform"</span>

<span class="token key atrule">TF_HTTP_LOCK_ADDRESS</span><span class="token punctuation">:</span> <span class="token string">"https://gitlab.com/api/v4/projects/${CI_PROJECT_ID}/terraform/state/hello-terraform/lock"</span>

<span class="token key atrule">TF_HTTP_UNLOCK_ADDRESS</span><span class="token punctuation">:</span> <span class="token string">"https://gitlab.com/api/v4/projects/${CI_PROJECT_ID}/terraform/state/hello-terraform/lock"</span>

<span class="token key atrule">TF_HTTP_USERNAME</span><span class="token punctuation">:</span> <span class="token string">"gitlab-ci-token"</span>

<span class="token key atrule">TF_HTTP_PASSWORD</span><span class="token punctuation">:</span> $CI_JOB_TOKEN

<span class="token key atrule">script</span><span class="token punctuation">:</span>

<span class="token punctuation">-</span> cd $TF_DIR

<span class="token punctuation">-</span> terraform init

<span class="token punctuation">-</span> terraform validate

<span class="token punctuation">-</span> terraform fmt <span class="token punctuation">-</span>check

<span class="token key atrule">rules</span><span class="token punctuation">:</span>

<span class="token punctuation">-</span> <span class="token key atrule">if</span><span class="token punctuation">:</span> $CI_MERGE_REQUEST_ID

<span class="token punctuation">-</span> <span class="token key atrule">if</span><span class="token punctuation">:</span> $CI_COMMIT_BRANCH == "main"

</code></pre>
<p><code>CI_PROJECT_ID</code> y <code>CI_JOB_TOKEN</code> son variables que GitLab inyecta automáticamente. No tenés que configurar nada en CI/CD Variables para esto.</p>
<hr>
<h2 id="part-2-—-aws-oidc-conexión-sin-access-keys">Part 2 — AWS OIDC (Conexión sin Access Keys)</h2>
<p>Con OIDC, GitLab le presenta un token JWT a AWS diciendo “soy el pipeline del repo X, branch Y”. AWS valida el token y entrega credenciales temporales (duran 1 hora). No hay secretos que guardar, rotar ni que se puedan filtrar.</p>
<h3 id="paso-2.1-—-creá-infraoidc.tf">Paso 2.1 — Creá <code>infra/oidc.tf</code></h3>
<pre class=" language-hcl"><code class="prism  language-hcl">
# Le dice a AWS: "confiá en tokens que vienen de GitLab"

resource "aws_iam_openid_connect_provider" "gitlab" {

url = "https://gitlab.com"

  

client_id_list = ["https://gitlab.com"]

thumbprint_list = ["b3dd7606d2b5a8b4a13771dbecc9ee1cecafa38a"]

  

tags = { Project = "hello-terraform" }

}

  

# Rol que GitLab asume via OIDC

resource "aws_iam_role" "gitlab_ci" {

name = "gitlab-ci-hello-terraform"

  

assume_role_policy = jsonencode({

Version = "2012-10-17"

Statement = [

{

Effect = "Allow"

Principal = {

Federated = aws_iam_openid_connect_provider.gitlab.arn

}

Action = "sts:AssumeRoleWithWebIdentity"

Condition = {

StringEquals = {

# Cambiá por tu namespace/proyecto real

"gitlab.com:sub" = "project_path:TU_NAMESPACE/TU_PROYECTO:ref_type:branch:ref:main"

}

}

}

]

})

  

tags = { Project = "hello-terraform" }

}

  

# Permisos: solo subir/borrar en el bucket del website

resource "aws_iam_role_policy" "gitlab_ci_s3" {

name = "deploy-to-website-bucket"

role = aws_iam_role.gitlab_ci.id

  

policy = jsonencode({

Version = "2012-10-17"

Statement = [

{

Effect = "Allow"

Action = [

"s3:PutObject",

"s3:DeleteObject",

"s3:ListBucket"

]

Resource = [

aws_s3_bucket.website.arn,

"${aws_s3_bucket.website.arn}/*"

]

}

]

})

}

</code></pre>
<p>La parte clave es el <code>Condition</code>. Solo el branch <code>main</code> de tu repo específico puede asumir este rol. Si alguien forkea tu proyecto, no funciona.</p>
<h3 id="paso-2.2-—-agregá-el-output">Paso 2.2 — Agregá el output</h3>
<p>En <code>infra/outputs.tf</code>:</p>
<pre class=" language-hcl"><code class="prism  language-hcl">
output "gitlab_ci_role_arn" {

value = aws_iam_role.gitlab_ci.arn

}

</code></pre>
<h3 id="paso-2.3-—-aplicá-los-cambios">Paso 2.3 — Aplicá los cambios</h3>
<p>Todavía estás usando las access keys locales, así que esto funciona normal:</p>
<pre class=" language-bash"><code class="prism  language-bash">
<span class="token function">cd</span>  infra

  

terraform  init  \

-backend-config<span class="token operator">=</span><span class="token string">"address=https://gitlab.com/api/v4/projects/TU_PROJECT_ID/terraform/state/hello-terraform"</span> \

-backend-config<span class="token operator">=</span><span class="token string">"lock_address=https://gitlab.com/api/v4/projects/TU_PROJECT_ID/terraform/state/hello-terraform/lock"</span>  \

-backend-config<span class="token operator">=</span><span class="token string">"unlock_address=https://gitlab.com/api/v4/projects/TU_PROJECT_ID/terraform/state/hello-terraform/lock"</span> \

-backend-config<span class="token operator">=</span><span class="token string">"username=TU_USUARIO_GITLAB"</span>  \

-backend-config<span class="token operator">=</span><span class="token string">"password=TU_PERSONAL_ACCESS_TOKEN"</span>

  

terraform  plan  -var<span class="token operator">=</span><span class="token string">"bucket_name=mi-hello-terraform-2026"</span>

terraform  apply  -var<span class="token operator">=</span><span class="token string">"bucket_name=mi-hello-terraform-2026"</span>

</code></pre>
<p>Copiá el <code>gitlab_ci_role_arn</code> del output.</p>
<h3 id="paso-2.4-—-configurá-la-variable-en-gitlab">Paso 2.4 — Configurá la variable en GitLab</h3>
<p><strong>Settings → CI/CD → Variables:</strong></p>
<p>| Key | Value | Protected | Masked |</p>
<p>|-----|-------|-----------|--------|</p>
<p>| <code>ROLE_ARN</code> | El ARN que copiaste del output | No | No |</p>
<h3 id="paso-2.5-—-reemplazá-.gitlab-ci.yml-completo">Paso 2.5 — Reemplazá <code>.gitlab-ci.yml</code> completo</h3>
<pre class=" language-yaml"><code class="prism  language-yaml">
<span class="token key atrule">stages</span><span class="token punctuation">:</span>

<span class="token punctuation">-</span> validate

<span class="token punctuation">-</span> deploy

  

<span class="token key atrule">variables</span><span class="token punctuation">:</span>

<span class="token key atrule">AWS_DEFAULT_REGION</span><span class="token punctuation">:</span> <span class="token string">"us-east-1"</span>

<span class="token key atrule">TF_DIR</span><span class="token punctuation">:</span> <span class="token string">"infra"</span>

  

<span class="token key atrule">validate</span><span class="token punctuation">:</span>

<span class="token key atrule">stage</span><span class="token punctuation">:</span> validate

<span class="token key atrule">image</span><span class="token punctuation">:</span>

<span class="token key atrule">name</span><span class="token punctuation">:</span> hashicorp/terraform<span class="token punctuation">:</span><span class="token number">1.7</span>

<span class="token key atrule">entrypoint</span><span class="token punctuation">:</span> <span class="token punctuation">[</span><span class="token string">""</span><span class="token punctuation">]</span>

<span class="token key atrule">variables</span><span class="token punctuation">:</span>

<span class="token key atrule">TF_HTTP_ADDRESS</span><span class="token punctuation">:</span> <span class="token string">"https://gitlab.com/api/v4/projects/${CI_PROJECT_ID}/terraform/state/hello-terraform"</span>

<span class="token key atrule">TF_HTTP_LOCK_ADDRESS</span><span class="token punctuation">:</span> <span class="token string">"https://gitlab.com/api/v4/projects/${CI_PROJECT_ID}/terraform/state/hello-terraform/lock"</span>

<span class="token key atrule">TF_HTTP_UNLOCK_ADDRESS</span><span class="token punctuation">:</span> <span class="token string">"https://gitlab.com/api/v4/projects/${CI_PROJECT_ID}/terraform/state/hello-terraform/lock"</span>

<span class="token key atrule">TF_HTTP_USERNAME</span><span class="token punctuation">:</span> <span class="token string">"gitlab-ci-token"</span>

<span class="token key atrule">TF_HTTP_PASSWORD</span><span class="token punctuation">:</span> $CI_JOB_TOKEN

<span class="token key atrule">script</span><span class="token punctuation">:</span>

<span class="token punctuation">-</span> cd $TF_DIR

<span class="token punctuation">-</span> terraform init

<span class="token punctuation">-</span> terraform validate

<span class="token punctuation">-</span> terraform fmt <span class="token punctuation">-</span>check

<span class="token key atrule">rules</span><span class="token punctuation">:</span>

<span class="token punctuation">-</span> <span class="token key atrule">if</span><span class="token punctuation">:</span> $CI_MERGE_REQUEST_ID

<span class="token punctuation">-</span> <span class="token key atrule">if</span><span class="token punctuation">:</span> $CI_COMMIT_BRANCH == "main"

  

<span class="token key atrule">deploy</span><span class="token punctuation">:</span>

<span class="token key atrule">stage</span><span class="token punctuation">:</span> deploy

<span class="token key atrule">image</span><span class="token punctuation">:</span>

<span class="token key atrule">name</span><span class="token punctuation">:</span> amazon/aws<span class="token punctuation">-</span>cli<span class="token punctuation">:</span>2.15.0

<span class="token key atrule">entrypoint</span><span class="token punctuation">:</span> <span class="token punctuation">[</span><span class="token string">""</span><span class="token punctuation">]</span>

<span class="token key atrule">id_tokens</span><span class="token punctuation">:</span>

<span class="token key atrule">GITLAB_OIDC_TOKEN</span><span class="token punctuation">:</span>

<span class="token key atrule">aud</span><span class="token punctuation">:</span> https<span class="token punctuation">:</span>//gitlab.com

<span class="token key atrule">before_script</span><span class="token punctuation">:</span>

<span class="token punctuation">-</span> <span class="token punctuation">&gt;</span>

STS_OUTPUT=$(aws sts assume<span class="token punctuation">-</span>role<span class="token punctuation">-</span>with<span class="token punctuation">-</span>web<span class="token punctuation">-</span>identity

<span class="token punctuation">-</span><span class="token punctuation">-</span>role<span class="token punctuation">-</span>arn $ROLE_ARN

<span class="token punctuation">-</span><span class="token punctuation">-</span>role<span class="token punctuation">-</span>session<span class="token punctuation">-</span>name "gitlab<span class="token punctuation">-</span>ci<span class="token punctuation">-</span>$<span class="token punctuation">{</span>CI_PIPELINE_ID<span class="token punctuation">}</span>"

<span class="token punctuation">-</span><span class="token punctuation">-</span>web<span class="token punctuation">-</span>identity<span class="token punctuation">-</span>token $GITLAB_OIDC_TOKEN

<span class="token punctuation">-</span><span class="token punctuation">-</span>duration<span class="token punctuation">-</span>seconds 3600

<span class="token punctuation">-</span><span class="token punctuation">-</span>output json)

<span class="token punctuation">-</span> export AWS_ACCESS_KEY_ID=$(echo $STS_OUTPUT <span class="token punctuation">|</span> grep <span class="token punctuation">-</span>o '"AccessKeyId"<span class="token punctuation">:</span> <span class="token string">"[^"</span><span class="token punctuation">]</span>*"'  <span class="token punctuation">|</span> cut <span class="token punctuation">-</span>d'"' <span class="token punctuation">-</span>f4)

<span class="token punctuation">-</span> export AWS_SECRET_ACCESS_KEY=$(echo $STS_OUTPUT <span class="token punctuation">|</span> grep <span class="token punctuation">-</span>o '"SecretAccessKey"<span class="token punctuation">:</span> <span class="token string">"[^"</span><span class="token punctuation">]</span>*"' <span class="token punctuation">|</span> cut <span class="token punctuation">-</span>d'"' <span class="token punctuation">-</span>f4)

<span class="token punctuation">-</span> export AWS_SESSION_TOKEN=$(echo $STS_OUTPUT <span class="token punctuation">|</span> grep <span class="token punctuation">-</span>o '"SessionToken"<span class="token punctuation">:</span> <span class="token string">"[^"</span><span class="token punctuation">]</span>*"' <span class="token punctuation">|</span> cut <span class="token punctuation">-</span>d'"' <span class="token punctuation">-</span>f4)

<span class="token key atrule">script</span><span class="token punctuation">:</span>

<span class="token punctuation">-</span> sed <span class="token punctuation">-</span>i "s/__TIMESTAMP__/$(date <span class="token punctuation">-</span>u)/" site/index.html

<span class="token punctuation">-</span> aws s3 sync site/ s3<span class="token punctuation">:</span>//$S3_BUCKET_NAME <span class="token punctuation">-</span><span class="token punctuation">-</span>delete

<span class="token key atrule">rules</span><span class="token punctuation">:</span>

<span class="token punctuation">-</span> <span class="token key atrule">if</span><span class="token punctuation">:</span> $CI_COMMIT_BRANCH == "main"

</code></pre>
<p>El bloque <code>id_tokens</code> le dice a GitLab que genere un JWT. En <code>before_script</code> se lo pasamos a AWS con <code>sts assume-role-with-web-identity</code> y obtenemos credenciales temporales.</p>
<h3 id="paso-2.6-—-pusheá-y-probá">Paso 2.6 — Pusheá y probá</h3>
<pre class=" language-bash"><code class="prism  language-bash">
<span class="token function">git</span>  add  <span class="token keyword">.</span>

<span class="token function">git</span>  commit  -m  <span class="token string">"Add GitLab state backend and OIDC"</span>

<span class="token function">git</span>  push  origin  main

</code></pre>
<p>Si el deploy pasa, OIDC está funcionando.</p>
<h3 id="paso-2.7-—-limpiá-las-access-keys">Paso 2.7 — Limpiá las access keys</h3>
<p>Una vez confirmado que funciona:</p>
<ol>
<li>
<p>En GitLab → Settings → CI/CD → Variables → <strong>borrá</strong>  <code>AWS_ACCESS_KEY_ID</code> y <code>AWS_SECRET_ACCESS_KEY</code></p>
</li>
<li>
<p>En AWS Console → IAM → Users → gitlab-deployer → Security credentials → <strong>Delete access keys</strong></p>
</li>
<li>
<p>Opcionalmente eliminá el usuario <code>gitlab-deployer</code> y su policy de <code>iam.tf</code></p>
</li>
</ol>
<hr>
<h2 id="estructura-final">Estructura final</h2>
<pre><code>
hello-terraform/

├── site/

│ └── index.html

├── infra/

│ ├── main.tf # S3 website + backend HTTP

│ ├── iam.tf # (podés limpiar el user viejo)

│ ├── oidc.tf # OIDC provider + IAM role

│ ├── variables.tf

│ └── outputs.tf

├── .gitlab-ci.yml

└── .gitignore

</code></pre>
<hr>
<h2 id="troubleshooting">Troubleshooting</h2>
<p><strong><code>Error acquiring the state lock</code>.</strong> Alguien (o un pipeline anterior que falló) dejó el state lockeado. Esperá unos minutos o hacé <code>terraform force-unlock LOCK_ID</code> con el ID que te muestra el error.</p>
<p><strong><code>AccessDenied</code> en el assume-role.</strong> Verificá que el <code>sub</code> en la Condition de <code>oidc.tf</code> coincida exactamente con tu namespace y proyecto. Un typo y AWS rechaza el token.</p>
<p><strong><code>Not authorized to perform sts:AssumeRoleWithWebIdentity</code>.</strong> El OIDC provider no se creó correctamente o el thumbprint cambió. Verificá en AWS Console → IAM → Identity providers.</p>
<p><strong>El validate falla con <code>backend initialization required</code>.</strong> Las variables <code>TF_HTTP_*</code> no se están inyectando. Revisá que estén en el bloque <code>variables</code> del job.</p>
<hr>
<h2 id="qué-cambió-respecto-a-la-parte-1">Qué cambió respecto a la parte 1</h2>
<p>| Antes | Ahora |</p>
<p>|-------|-------|</p>
<p>| State en tu máquina | State en GitLab con locking automático |</p>
<p>| Access keys estáticas en CI/CD Variables | Token OIDC → credenciales temporales (1 hora) |</p>
<p>| Si se filtran las keys, acceso permanente | Si se filtra el token, expira en minutos |</p>
<p>| Cualquier pipeline podía usar las keys | Solo el branch <code>main</code> de tu repo puede asumir el rol |</p>
<hr>
<h2 id="siguiente-paso">Siguiente paso</h2>
<p>Con el state centralizado y la conexión keyless, estamos listos para empezar a agregar infraestructura real: una app Laravel en EC2, con su IAM Role, Security Groups y RDS.</p>
<pre><code></code></pre>


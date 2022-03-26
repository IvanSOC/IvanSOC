mesh generator:

    private Mesh mesh;
    private GeneratorSettings generatorSettings;

    private Vector3[] vertices;
    private int[] triangles;

    private int worldSize;

    private Gradient groundGradient;
    private Color[] colors;

    private void Start()
    {
        mesh = new Mesh();
        GetComponent<MeshFilter>().mesh = mesh;

        generatorSettings = GameObject.FindGameObjectWithTag("MainCamera").GetComponent<GeneratorSettings>();

        worldSize = generatorSettings.worldSize;

        groundGradient = generatorSettings.groundGradient;

        CreateShape();
        UpdateMesh();
    }

    private void CreateShape()
    {
        vertices = new Vector3[(worldSize + 1) * (worldSize + 1)];
        colors = new Color[vertices.Length];

        for (int i = 0, z = 0; z <= worldSize; z++)
        {
            for (int x = 0; x <= worldSize; x++)
            {
                //colors[i] = gradient.Evaluate(Mathf.PerlinNoise(Random.Range(0, 100) * .5f, Random.Range(0, 100) * 5) * 1.1f);
                colors[i] = groundGradient.Evaluate(Random.Range(0f, 1f));
                vertices[i] = new Vector3(x, 0, z);
                i++;
            }
        }

        triangles = new int[worldSize * worldSize * 6];

        int vert = 0;
        int tris = 0;
        for (int z = 0; z < worldSize; z++)
        {
            for (int x = 0; x < worldSize; x++)
            {
                triangles[tris + 0] = vert + 0;
                triangles[tris + 1] = vert + worldSize + 1;
                triangles[tris + 2] = vert + 1;
                triangles[tris + 3] = vert + 1;
                triangles[tris + 4] = vert + worldSize + 1;
                triangles[tris + 5] = vert + worldSize + 2;

                vert++;
                tris += 6;
            }

            vert++;
        }
    }

    private void UpdateMesh()
    {
        mesh.vertices = vertices;
        mesh.triangles = triangles;
        mesh.colors = colors;

        mesh.RecalculateNormals();
    }

camera controller:

    private Camera cam;
    private PanelOpener panelOpener;
    private GeneratorSettings generatorSettings;

    private Vector3 startPosition;
    private Vector3 targetPosition;

    private float worldCenter;
    private float cameraCenter;
    private float nullPosition;

    private float positiveLimit;
    private float negativeLimit;

    private float mouseWheelSpeed = 10f;

    private void Start()
    {
        cam = GetComponent<Camera>();
        panelOpener = GetComponent<PanelOpener>();
        generatorSettings = GetComponent<GeneratorSettings>();

        SetCameraSettings();
    }

    private void Update()
    {
        if (Input.GetMouseButtonDown(2))
        {
            startPosition = cam.ScreenToWorldPoint(Input.mousePosition);
            for (int i = 0; i < panelOpener.panels.Length; i++)
            {
                panelOpener.panels[i].SetActive(false);
            }
        }
        else if (Input.GetMouseButton(2))
        {
            Vector3 position = transform.position;
            targetPosition = cam.ScreenToWorldPoint(Input.mousePosition) - startPosition;
            position.x = Mathf.Clamp(transform.position.x - targetPosition.x, negativeLimit, positiveLimit);
            position.z = Mathf.Clamp(transform.position.z - targetPosition.z, negativeLimit, positiveLimit);
            transform.position = position;
        }

        float mouseWheel = Input.GetAxis("Mouse ScrollWheel");
        if (mouseWheel != 0)
        {
            cam.orthographicSize = Mathf.Clamp(cam.orthographicSize + -mouseWheel * mouseWheelSpeed, 3, 22);
        }
    }

    private void SetCameraSettings()
    {
        worldCenter = (float)generatorSettings.worldSize / 2;
        cameraCenter = Mathf.Sqrt(Mathf.Pow(transform.position.y, 2) + Mathf.Pow(transform.position.y, 2));
        cameraCenter /= 2;
        nullPosition = worldCenter - cameraCenter;
        transform.position = new Vector3(nullPosition, transform.position.y, nullPosition);

        positiveLimit = nullPosition + worldCenter;
        negativeLimit = nullPosition - worldCenter;
    }

world generator:

    private GeneratorSettings generatorSettings;

    private int worldSize;
    private GameObject[] objectsToSpawn;
    private int[] objectsLimit;
    private int[] xObjectSize;
    private int[] zObjectSize;

    private int xSizeSpawnLimit;
    private int zSizeSpawnLimit;
    public bool[] occupiedPoints;
    private Quaternion rotation;

    private int position;
    private int option;

    private void Start()
    {
        GetSettings();
        GetObject();
    }

    private void GetSettings()
    {
        generatorSettings = GameObject.FindGameObjectWithTag("MainCamera").GetComponent<GeneratorSettings>();
        worldSize = generatorSettings.worldSize;

        objectsToSpawn = generatorSettings.objectsToSpawn;

        objectsLimit = generatorSettings.numberOfObjectsToSpawn;

        xObjectSize = generatorSettings.xObjectSize;
        zObjectSize = generatorSettings.zObjectSize;
        occupiedPoints = new bool[worldSize * worldSize];
    }

    private void GetObject()
    {
        for (option = 0; option < objectsToSpawn.Length; option++)
        {
            GetSpawnPermission();
        }
    }

    private void GetSpawnPermission()
    {
        while (objectsLimit[option] != 0)
        { 
            position = 0;
            for (int z = 0; z < worldSize; z++)
            {
                for (int x = 0; x < worldSize; x++)
                {
                    if (occupiedPoints[position] != true)
                    {
                        if (Random.Range(0, 10000) < 1)
                        {
                            if (xObjectSize[option] > 1 || zObjectSize[option] > 1)
                            {
                                xSizeSpawnLimit = x + xObjectSize[option];
                                zSizeSpawnLimit = z + zObjectSize[option];

                                if (--xSizeSpawnLimit < worldSize && --zSizeSpawnLimit < worldSize) OccupiedPointsCalculating(x, z);
                            }
                            else
                            {
                                occupiedPoints[position] = true;
                                SpawnObject(x, z);  
                            }

                            if (objectsLimit[option] == 0) return;
                        }
                    }

                    position++;
                }
            }
        }
    }

    private void OccupiedPointsCalculating(float xPosition, float zPosition)
    {
        int positionForCalculatingPoints;
        int[] savedPositions = new int[xObjectSize[option] * zObjectSize[option]];
        for (int i = 0, z = 0; z < zObjectSize[option]; z++)
        {
            positionForCalculatingPoints = position;
            positionForCalculatingPoints += worldSize * z;
            for (int x = 0; x < xObjectSize[option]; x++)
            {
                if (occupiedPoints[positionForCalculatingPoints] == true) return;
                savedPositions[i] = positionForCalculatingPoints;
                positionForCalculatingPoints++;
                i++;
            }
        }

        foreach (int savedPosition in savedPositions)
        {
            occupiedPoints[savedPosition] = true;
        }

        xPosition += .5f * xObjectSize[option] - .5f;
        zPosition += .5f * zObjectSize[option] - .5f;
        SpawnObject(xPosition, zPosition);
    }

    private void SpawnObject(float x, float z)
    {
        rotation = xObjectSize[option] == zObjectSize[option] ? Quaternion.Euler(new Vector3(0, Random.Range(0, 360), 0)) : Quaternion.identity;
        Instantiate(objectsToSpawn[option], new Vector3(x + .5f, objectsToSpawn[option].transform.position.y / 2, z + .5f), rotation);
        --objectsLimit[option];
    }
